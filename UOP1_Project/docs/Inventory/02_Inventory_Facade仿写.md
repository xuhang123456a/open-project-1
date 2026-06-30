# Inventory Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `InventorySO` 背包数据 | 保留核心逻辑 | ✅保留 |
| `ItemSO` 物品定义 | 保留核心逻辑 | ✅保留 |
| `ItemStack` 结构体 | 保留 | ✅保留 |
| `InventoryManager` 事件驱动 | 保留核心逻辑 | ✅保留 |
| `ItemPicker` 拾取器 | 保留 | ✅保留 |
| `CollectableItem` 可收集物品 | 简化 | ⚠️ 简化 |
| 配方系统 | 保留核心逻辑 | ✅保留 |
| 存档集成 | 简化 | ⚠️ 简化 |

---

## 最小复刻代码（~160行）

```csharp
using System.Collections.Generic;
using UnityEngine;

// ==================== 物品定义 ====================

public enum ItemType
{
    Consumable,
    Weapon,
    Material
}

[CreateAssetMenu(menuName = "Inventory/Item")]
public class ItemSO : ScriptableObject
{
    public string itemName;
    public Sprite icon;
    public ItemType itemType;
    public GameObject prefab;
    
    // 配方
    public List<ItemStack> ingredients;
    public ItemSO result;
}

// ==================== 物品堆叠 ====================

[System.Serializable]
public struct ItemStack
{
    public ItemSO item;
    public int amount;
    
    public ItemStack(ItemSO item, int amount)
    {
        this.item = item;
        this.amount = amount;
    }
}

// ==================== 背包数据 ====================

[CreateAssetMenu(menuName = "Inventory/Inventory")]
public class InventorySO : ScriptableObject
{
    [SerializeField] private List<ItemStack> _items = new List<ItemStack>();
    [SerializeField] private List<ItemStack> _defaultItems = new List<ItemStack>();
    
    public List<ItemStack> Items => _items;
    
    public void Init()
    {
        _items.Clear();
        foreach (var item in _defaultItems)
            _items.Add(new ItemStack(item.item, item.amount));
    }
    
    public void Add(ItemSO item, int count = 1)
    {
        if (count <= 0) return;
        
        for (int i = 0; i < _items.Count; i++)
        {
            if (_items[i].item == item)
            {
                // 只有可消耗品才堆叠
                if (item.itemType == ItemType.Consumable)
                    _items[i].amount += count;
                return;
            }
        }
        
        _items.Add(new ItemStack(item, count));
    }
    
    public void Remove(ItemSO item, int count = 1)
    {
        if (count <= 0) return;
        
        for (int i = 0; i < _items.Count; i++)
        {
            if (_items[i].item == item)
            {
                _items[i].amount -= count;
                if (_items[i].amount <= 0)
                    _items.RemoveAt(i);
                return;
            }
        }
    }
    
    public bool Contains(ItemSO item)
    {
        foreach (var stack in _items)
            if (stack.item == item) return true;
        return false;
    }
    
    public int Count(ItemSO item)
    {
        foreach (var stack in _items)
            if (stack.item == item) return stack.amount;
        return 0;
    }
    
    public bool HasIngredients(List<ItemStack> ingredients)
    {
        foreach (var required in ingredients)
        {
            if (Count(required.item) < required.amount)
                return false;
        }
        return true;
    }
}

// ==================== 背包管理器 ====================

public class InventoryManager : MonoBehaviour
{
    [SerializeField] private InventorySO _inventory;
    
    // 事件
    public System.Action<ItemSO> OnItemAdded;
    public System.Action<ItemSO> OnItemRemoved;
    
    public void AddItem(ItemSO item)
    {
        _inventory.Add(item);
        OnItemAdded?.Invoke(item);
    }
    
    public void RemoveItem(ItemSO item)
    {
        _inventory.Remove(item);
        OnItemRemoved?.Invoke(item);
    }
    
    public bool CookRecipe(ItemSO recipe)
    {
        if (!_inventory.Contains(recipe)) return false;
        if (!_inventory.HasIngredients(recipe.ingredients)) return false;
        
        // 消耗材料
        foreach (var ingredient in recipe.ingredients)
            _inventory.Remove(ingredient.item, ingredient.amount);
        
        // 添加产物
        _inventory.Add(recipe.result);
        return true;
    }
}

// ==================== 物品拾取器 ====================

public class ItemPicker : MonoBehaviour
{
    public void PickItem(ItemSO item)
    {
        // 通知 InventoryManager 添加物品
        FindObjectOfType<InventoryManager>()?.AddItem(item);
        Destroy(gameObject);
    }
}

// ==================== ===

public class PlayerInteraction : MonoBehaviour
{
    [SerializeField] private InventorySO _inventory;
    
    private void OnTriggerEnter(Collider other)
    {
        if (other.TryGetComponent(out ItemPicker picker))
        {
            // 拾取物品
        }
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| ItemStack 结构体 | ✅保留 | 轻量包装 |
| 可堆叠检查 | ✅保留 | 核心机制 |
| 配方材料检查 | ✅保留 | 核心机制 |
| 事件驱动 | ✅保留 | 核心机制 |
| Guid 序列化 | ❌砍掉 | 存档细节 |
| **最容易搞错** | ⚠️ | 忘记检查物品类型导致错误堆叠 |
