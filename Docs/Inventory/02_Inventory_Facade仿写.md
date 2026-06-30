# Inventory 模块 Facade 仿写

> 目标：复刻三个核心不变量——①类型决定是否堆叠（Use 才累加）；②归零移除的迭代安全；③`Init` 深拷贝防默认列表污染。
> 用纯 C# 表达，砍掉 ScriptableObject、本地化、UI、配方派生（保留配方查询的双 API 形态）。

---

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `InventorySO : ScriptableObject` | `Inventory` 普通类 | ⚠️ 去 SO，调用方持有实例 |
| `ItemSO : SerializableScriptableObject` | `Item`(Id + Type) | ⚠️ Id 模拟 GUID |
| `ItemTypeSO.ActionType` | `ItemActionType` 枚举 | ✅ 保留类型驱动 |
| `ItemStack`(class, Amount 可变) | 同（class）| ✅ 保留引用语义 |
| `Init` 深拷贝默认项 | 同 | ✅ 保留（核心）|
| `Add` Use 才堆叠 | 同 | ✅ 保留（核心）|
| `Remove` 归零移除 + return | 同 | ✅ 保留（核心迭代安全）|
| `IngredientsAvailability` / `hasIngredients` | 同两 API | ✅ 保留双形态 |
| 配方/本地化派生 | 删除 | ❌ 砍掉 |

---

## 最小复刻代码（独立可编译，~150 行）

```csharp
using System;
using System.Collections.Generic;

namespace MiniInventory
{
    public enum ItemActionType { Use, KeyItem, Cook }   // 对应 ItemInventoryActionType

    public sealed class Item
    {
        public string Id;                                // 模拟 GUID
        public string Name;
        public ItemActionType ActionType;
        public Item(string id, string name, ItemActionType type) { Id = id; Name = name; ActionType = type; }
    }

    public sealed class ItemStack
    {
        public Item Item { get; }                        // 只读：栈建立后物品不可换
        public int Amount;                               // 可变
        public ItemStack(Item item, int amount) { Item = item; Amount = amount; }
        public ItemStack(ItemStack other) { Item = other.Item; Amount = other.Amount; } // 深拷贝构造
    }

    public sealed class Inventory
    {
        private readonly List<ItemStack> _items = new();
        private readonly List<ItemStack> _defaultItems;
        public IReadOnlyList<ItemStack> Items => _items;

        public Inventory(List<ItemStack> defaultItems = null) => _defaultItems = defaultItems ?? new();

        // 深拷贝默认项，防止运行时增删污染配置数据
        public void Init()
        {
            _items.Clear();
            foreach (var d in _defaultItems)
                _items.Add(new ItemStack(d));            // new 拷贝，绝不直接放 d 的引用
        }

        public void Add(Item item, int count = 1)
        {
            if (count <= 0) return;
            for (int i = 0; i < _items.Count; i++)
            {
                var stack = _items[i];
                if (stack.Item == item)
                {
                    // 类型决定是否堆叠：只有 Use 才累加，关键道具不增量
                    if (stack.Item.ActionType == ItemActionType.Use)
                        stack.Amount += count;
                    return;                              // 命中即结束（无论是否堆叠）
                }
            }
            _items.Add(new ItemStack(item, count));      // 未命中：新建栈
        }

        public void Remove(Item item, int count = 1)
        {
            if (count <= 0) return;
            for (int i = 0; i < _items.Count; i++)
            {
                var stack = _items[i];
                if (stack.Item == item)
                {
                    stack.Amount -= count;
                    if (stack.Amount <= 0)
                        _items.Remove(stack);            // 归零移除
                    return;                              // 立即 return → 迭代安全（不再访问失效索引）
                }
            }
        }

        public bool Contains(Item item)
        {
            for (int i = 0; i < _items.Count; i++)
                if (_items[i].Item == item) return true;
            return false;
        }

        public int Count(Item item)
        {
            for (int i = 0; i < _items.Count; i++)
                if (_items[i].Item == item) return _items[i].Amount;
            return 0;
        }

        // 逐原料是否齐备（供 UI 显示缺哪个）
        public bool[] IngredientsAvailability(List<ItemStack> ingredients)
        {
            if (ingredients == null) return null;
            var result = new bool[ingredients.Count];
            for (int i = 0; i < ingredients.Count; i++)
                result[i] = _items.Exists(o => o.Item == ingredients[i].Item && o.Amount >= ingredients[i].Amount);
            return result;
        }

        // 整体能否合成（双否定：不存在任一缺料）
        public bool HasIngredients(List<ItemStack> ingredients)
            => !ingredients.Exists(need => !_items.Exists(have => have.Item == need.Item && have.Amount >= need.Amount));
    }
}
```

---

## 使用示例

```csharp
using System;
using System.Collections.Generic;
using MiniInventory;

class Demo
{
    static void Main()
    {
        var apple = new Item("apple", "Apple", ItemActionType.Use);
        var key   = new Item("key", "Boss Key", ItemActionType.KeyItem);

        var defaults = new List<ItemStack> { new ItemStack(apple, 1) };
        var inv = new Inventory(defaults);
        inv.Init();                       // 深拷贝默认：背包有 Apple x1（不影响 defaults）

        inv.Add(apple, 3);                // Use 类型 → 堆叠 → Apple x4
        inv.Add(key);                     // 新建 Boss Key x1
        inv.Add(key);                     // KeyItem 命中但不堆叠 → 仍 x1
        Console.WriteLine($"Apple={inv.Count(apple)}, Key={inv.Count(key)}"); // Apple=4, Key=1

        inv.Remove(apple, 4);             // 归零 → 从列表移除
        Console.WriteLine($"含 Apple? {inv.Contains(apple)}");                // false

        // 配方查询
        var recipe = new List<ItemStack> { new ItemStack(key, 1) };
        Console.WriteLine($"可合成? {inv.HasIngredients(recipe)}");           // true（有 Key）

        // 验证 Init 未污染 defaults：再次 Init 仍是 Apple x1
        inv.Init();
        Console.WriteLine($"重置后 Apple={inv.Count(apple)}");                // 1
    }
}
```

---

## 取舍自检

- ✅ **保留**：类型驱动的堆叠规则（Use 才累加）、`Remove` 归零移除 + 立即 return 的迭代安全、`Init` 深拷贝防默认列表污染、`ItemStack` class 引用语义（取出即可改 Amount）、`Item` 只读、配方查询双 API（逐项 bool[] + 整体 bool 双否定）。
- ❌ **砍掉**：ScriptableObject 资产化、`SerializableScriptableObject` 真实 GUID（用 Id 模拟）、本地化 `LocalizedString`、`ItemRecipeSO` 配方派生与 `virtual` 扩展、UI/Picker 表现层、事件通道通知。
- ⚠️ **最容易搞错的一处**：`Init` 必须 `new ItemStack(d)` **深拷贝**默认项。若写成 `_items.Add(d)` 直接放默认项引用，运行时 `Add`/`Remove` 改的是 `_defaultItems` 里的同一个 ItemStack（class 引用共享）——「新游戏初始内容」被运行时永久改写，下次 `Init` 就拿到被污染的默认值。其次：`Remove` 的「遍历中移除 + 立即 return」只对「单物品操作」安全；若扩展成批量移除多种物品，必须改反向 for / `RemoveAll`，否则迭代器失效或跳过元素。
