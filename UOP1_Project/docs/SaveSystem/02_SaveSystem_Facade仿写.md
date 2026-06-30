# SaveSystem Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `SaveSystem` 存档管理器 | 保留核心逻辑 | ✅保留 |
| `Save` 存档数据结构 | 保留 | ✅保留 |
| `FileManager` 文件IO | 保留 | ✅保留 |
| `SerializedItemStack` | 保留 | ✅保留 |
| 备份机制 | 保留 | ✅保留 |
| Addressables加载 | 简化为直接加载 | ⚠️ 简化 |
| 设置保存 | 简化 | ⚠️ 简化 |

---

## 最小复刻代码（~120行）

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using UnityEngine;

// ==================== 存档数据 ====================

[Serializable]
public class Save
{
    public string locationId;
    public List<SerializedItemStack> itemStacks = new List<SerializedItemStack>();
    public List<string> questlineItemGuids = new List<string>();
    
    public string ToJson() => JsonUtility.ToJson(this);
    
    public void LoadFromJson(string json)
    {
        JsonUtility.FromJsonOverwrite(json, this);
    }
}

[Serializable]
public class SerializedItemStack
{
    public string itemGuid;
    public int amount;
    
    public SerializedItemStack(string guid, int amount)
    {
        this.itemGuid = guid;
        this.amount = amount;
    }
}

// ==================== 文件管理 ====================

public static class FileManager
{
    public static bool WriteToFile(string fileName, string content)
    {
        var path = Path.Combine(Application.persistentDataPath, fileName);
        try
        {
            File.WriteAllText(path, content);
            return true;
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to write: {e}");
            return false;
        }
    }
    
    public static bool LoadFromFile(string fileName, out string result)
    {
        var path = Path.Combine(Application.persistentDataPath, fileName);
        if (!File.Exists(path))
        {
            result = "";
            return false;
        }
        try
        {
            result = File.ReadAllText(path);
            return true;
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to read: {e}");
            result = "";
            return false;
        }
    }
    
    public static bool MoveFile(string oldName, string newName)
    {
        var oldPath = Path.Combine(Application.persistentDataPath, oldName);
        var newPath = Path.Combine(Application.persistentDataPath, newName);
        try
        {
            if (File.Exists(newPath)) File.Delete(newPath);
            if (File.Exists(oldPath)) File.Move(oldPath, newPath);
            return true;
        }
        catch (Exception e)
        {
            Debug.LogError($"Failed to move: {e}");
            return false;
        }
    }
}

// ==================== 存档系统 ====================

public class SaveSystem : ScriptableObject
{
    public string saveFileName = "save.json";
    public string backupFileName = "save.json.bak";
    public Save saveData = new Save();
    
    public bool LoadFromFile()
    {
        if (FileManager.LoadFromFile(saveFileName, out string json))
        {
            saveData.LoadFromJson(json);
            return true;
        }
        return false;
    }
    
    public void SaveToFile()
    {
        // 备份
        FileManager.MoveFile(saveFileName, backupFileName);
        // 写入
        FileManager.WriteToFile(saveFileName, saveData.ToJson());
    }
    
    public void SetNewGame()
    {
        saveData = new Save();
        SaveToFile();
    }
}

// ==================== 使用示例 ====================

public class GameSaver : MonoBehaviour
{
    [SerializeField] private SaveSystem _saveSystem;
    [SerializeField] private InventorySO _inventory;
    
    public void SaveGame()
    {
        _saveSystem.saveData.itemStacks.Clear();
        foreach (var item in _inventory.Items)
saveSystem.saveData.itemStacks.Add(
                new SerializedItemStack(item.item.Guid, item.amount));
        }
        _saveSystem.SaveToFile();
    }
    
    public void LoadGame()
    {
        if (_saveSystem.LoadFromFile())
        {
            _inventory.Init();
            foreach (var saved in _saveSystem.saveData.itemStacks)
            {
                // 通过Guid加载ItemSO并添加到背包
                // var item = LoadItemByGuid(saved.itemGuid);
                // _inventory.Add(item, saved.amount);
            }
        }
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| JSON序列化 | ✅保留 | 核心机制 |
| Guid引用 | ✅保留 | 核心机制 |
| 备份机制 | ✅保留 | 核心机制 |
| 文件IO | ✅保留 | 核心机制 |
| Addressables | ❌砍掉 | 平台细节 |
| **最容易搞错** | ⚠️ | 忘记备份直接写入，可能导致存档损坏 |
