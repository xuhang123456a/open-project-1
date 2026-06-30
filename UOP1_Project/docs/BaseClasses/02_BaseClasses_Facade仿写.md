# BaseClasses Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `DescriptionBaseSO` | 保留 | ✅保留 |
| `SerializableScriptableObject` | 保留核心逻辑 | ✅保留 |
| `OnValidate` Guid 生成 | 保留 | ✅保留 |

---

## 最小复刻代码（~40行）

```csharp
using UnityEngine;

// ==================== 描述基类 ====================

public class DescriptionBaseSO : ScriptableObject
{
    [TextArea] public string description;
}

// ==================== 可序列化基类 ====================

public class SerializableScriptableObject : ScriptableObject
{
    [SerializeField, HideInInspector] private string _guid;
    public string Guid => _guid;

#if UNITY_EDITOR
    void OnValidate()
    {
        var path = UnityEditor.AssetDatabase.GetAssetPath(this);
        _guid = UnityEditor.AssetDatabase.AssetPathToGUID(path);
    }
#endif
}

// ==================== 使用示例 ====================

[CreateAssetMenu(menuName = "Items/Item")]
public class ItemSO : SerializableScriptableObject
{
    public string itemName;
    public Sprite icon;
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 描述字段 | ✅保留 | Inspector 注释需要 |
| Guid 序列化 | ✅保留 | 存档系统需要 |
| **最容易搞错** | ⚠️ | Guid 只在 Editor 下生成，运行时不会更新 |
