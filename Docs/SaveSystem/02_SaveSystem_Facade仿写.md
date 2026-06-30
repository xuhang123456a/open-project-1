# SaveSystem 模块 Facade 仿写

> 目标：复刻三个核心不变量——①GUID 作为「引用持久化」稳定键；②富对象↔扁平 DTO 对称转换；③写时备份容错。
> 用纯 C# + `System.Text.Json`（或手写 JSON 占位）模拟，砍掉 Addressables（用自建 GUID 注册表替代）、协程、事件订阅。

---

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `SerializableScriptableObject._guid` + OnValidate | `IGuidAsset` { Guid } + 注册表 | ⚠️ 用注册表替 AssetDatabase |
| `Addressables.LoadAssetAsync<T>(guid)` | `AssetRegistry.Resolve(guid)` 同步 | ⚠️ 同步替异步协程 |
| `Save`(JsonUtility) | `SaveData` + `System.Text.Json` | ✅ 保留 DTO + 序列化 |
| `SerializedItemStack`(guid+amount) | 同 | ✅ 保留 |
| `FileManager`(try-catch bool) | 同 | ✅ 保留容错风格 |
| `MoveFile` 写时备份 | 同 | ✅ 保留（核心容错）|
| 富对象→DTO 显式转换 | 同（Capture/Restore）| ✅ 保留对称转换 |
| 事件订阅触发 | 删除，改显式 Save()/Load() | ❌ 砍掉 |

---

## 最小复刻代码（独立可编译，~170 行）

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Text.Json;

namespace MiniSave
{
    // ---------- GUID 作为引用持久化的稳定键 ----------
    public interface IGuidAsset { string Guid { get; } }

    public sealed class ItemAsset : IGuidAsset
    {
        public string Guid { get; }
        public string DisplayName;
        public ItemAsset(string guid, string name) { Guid = guid; DisplayName = name; }
    }

    // 替代 Addressables：GUID → 资产 的注册表（编辑期/启动期填充）
    public sealed class AssetRegistry
    {
        private readonly Dictionary<string, IGuidAsset> _byGuid = new();
        public void Register(IGuidAsset a) => _byGuid[a.Guid] = a;
        public T Resolve<T>(string guid) where T : class, IGuidAsset
            => _byGuid.TryGetValue(guid, out var a) ? a as T : null;
    }

    // ---------- 内存富对象 ----------
    public sealed class ItemStack { public ItemAsset Item; public int Amount; }
    public sealed class Inventory { public List<ItemStack> Items = new(); }

    // ---------- 磁盘扁平 DTO ----------
    public sealed class SerializedItemStack { public string ItemGuid { get; set; } public int Amount { get; set; } }
    public sealed class SaveData
    {
        public string LocationId { get; set; }
        public List<SerializedItemStack> ItemStacks { get; set; } = new();
        public float MasterVolume { get; set; }
        public string ToJson() => JsonSerializer.Serialize(this);
        public static SaveData FromJson(string json) =>
            string.IsNullOrWhiteSpace(json) ? new SaveData() : JsonSerializer.Deserialize<SaveData>(json);
    }

    // ---------- 文件 IO：全部 try-catch 返回 bool，不抛上层 ----------
    public static class FileManager
    {
        public static bool Write(string path, string contents)
        {
            try { File.WriteAllText(path, contents); return true; }
            catch (Exception e) { Console.WriteLine($"[IO] write fail: {e.Message}"); return false; }
        }
        public static bool Load(string path, out string result)
        {
            try { if (!File.Exists(path)) File.WriteAllText(path, ""); result = File.ReadAllText(path); return true; }
            catch (Exception e) { Console.WriteLine($"[IO] read fail: {e.Message}"); result = ""; return false; }
        }
        // 写时备份：现档 → .bak（崩溃时旧档仍在）
        public static bool Backup(string path, string bakPath)
        {
            try
            {
                if (File.Exists(bakPath)) File.Delete(bakPath);
                if (!File.Exists(path)) return false;
                File.Move(path, bakPath);
                return true;
            }
            catch (Exception e) { Console.WriteLine($"[IO] backup fail: {e.Message}"); return false; }
        }
    }

    // ---------- 存档中枢：富对象↔DTO 双向翻译 ----------
    public sealed class SaveSystem
    {
        private readonly AssetRegistry _registry;
        private readonly Inventory _inventory;
        private readonly string _file, _bak;
        public SaveData Data = new();

        public SaveSystem(AssetRegistry registry, Inventory inv, string dir)
        {
            _registry = registry; _inventory = inv;
            _file = Path.Combine(dir, "save.chop");
            _bak = Path.Combine(dir, "save.chop.bak");
        }

        public void SaveToDisk(string locationGuid)
        {
            // 富对象 → DTO（清空重填，必须穷尽所有状态）
            Data.LocationId = locationGuid;
            Data.ItemStacks.Clear();
            foreach (var s in _inventory.Items)
                Data.ItemStacks.Add(new SerializedItemStack { ItemGuid = s.Item.Guid, Amount = s.Amount });

            // 写时备份再写新档
            FileManager.Backup(_file, _bak);
            FileManager.Write(_file, Data.ToJson());
        }

        public bool LoadFromDisk()
        {
            if (!FileManager.Load(_file, out var json)) return false;
            Data = SaveData.FromJson(json);
            return true;
        }

        // 反水合：DTO → 富对象（GUID 经注册表反查重建引用）
        public void RehydrateInventory()
        {
            _inventory.Items.Clear();
            foreach (var ss in Data.ItemStacks)
            {
                var item = _registry.Resolve<ItemAsset>(ss.ItemGuid);
                if (item != null) _inventory.Items.Add(new ItemStack { Item = item, Amount = ss.Amount });
                else Console.WriteLine($"[Save] GUID {ss.ItemGuid} 无法解析，物品丢失");
            }
        }

        public void NewGame() { FileManager.Write(_file, ""); _inventory.Items.Clear(); Data = new SaveData(); }
    }
}
```

---

## 使用示例

```csharp
using System;
using System.IO;
using MiniSave;

class Demo
{
    static void Main()
    {
        var dir = Path.Combine(Path.GetTempPath(), "minisave_demo");
        Directory.CreateDirectory(dir);

        // 启动期：注册可被引用的资产（GUID 稳定键）
        var registry = new AssetRegistry();
        var apple = new ItemAsset("guid-apple-001", "Apple");
        var key   = new ItemAsset("guid-key-002",   "Rusty Key");
        registry.Register(apple); registry.Register(key);

        var inv = new Inventory();
        inv.Items.Add(new ItemStack { Item = apple, Amount = 3 });
        inv.Items.Add(new ItemStack { Item = key, Amount = 1 });

        var save = new SaveSystem(registry, inv, dir);
        save.SaveToDisk(locationGuid: "guid-forest-100");   // 富对象→DTO→落盘(+备份)

        // 模拟重启：新进程读盘 + 反水合
        var inv2 = new Inventory();
        var save2 = new SaveSystem(registry, inv2, dir);
        save2.LoadFromDisk();
        save2.RehydrateInventory();
        foreach (var s in inv2.Items)
            Console.WriteLine($"恢复 {s.Item.DisplayName} x{s.Amount}");   // Apple x3 / Rusty Key x1
        Console.WriteLine($"上次位置 GUID = {save2.Data.LocationId}");
    }
}
```

---

## 取舍自检

- ✅ **保留**：GUID 作引用持久化稳定键 + 注册表反查、富对象↔扁平 DTO 的对称转换（Clear 重填）、写时备份容错、文件 IO 全 try-catch 返回 bool 不抛上层、读文件不存在时建空文件、JSON 序列化。
- ❌ **砍掉**：`SerializableScriptableObject.OnValidate` 的 `AssetDatabase` GUID 固化（改手动注册）、Addressables 异步加载 + 协程（改同步 Resolve）、事件订阅触发（改显式调用）、Quest/Settings/Locale 等其余存档字段。
- ⚠️ **最容易搞错的一处**：富对象→DTO 转换里**每次都 `Clear()` 再重填**。若新增了一类运行时状态（如 Buff）却忘了在 `SaveToDisk` 加进 DTO，存档会**静默丢失**该状态且不报错（加载时该字段为默认值）。保存写入字段集与加载读取字段集必须严格对称。其次：绝不在唯一存档上原地 `WriteAllText` 覆盖——必须先 `Backup`，否则一次写入中断就永久损坏存档。
