# Inventory 🟢 概念题

**Q1: `ItemStack` 为什么是 struct 而非 class？**

<details>
<summary>参考答案</summary>

轻量包装，避免堆分配，减少 GC 压力。
</details>

---

**Q2: `InventorySO.Add()` 中为什么要检查 `ItemType.Use`？**

<details>
<summary>参考答案</summary>

只有可消耗品才堆叠，武器等不应堆叠。
</details>

---

**Q3: `hasIngredients()` 是如何检查配方的？**

<details>
<summary>参考答案</summary>

遍历所有材料，检查背包中每种材料的数量是否足够。
</details>

---

**Q4: `ItemSO` 为什么要继承 `SerializableScriptableObject`？**

<details>
<summary>参考答案</summary>

支持 Guid 序列化，用于存档系统。
</details>

---

**Q5: `InventoryManager` 为什么要监听事件而非直接调用？**

<details>
<summary>参考答案</summary>

解耦，支持多个系统触发相同操作（如拾取、任务奖励、商店购买）。
</details>

---

## 🟡 机制题

**Q6: 如果添加一个已存在的不可堆叠物品，会发生什么？**

<details>
<summary>参考答案</summary>

忽略，不添加也不增加数量。
</details>

---

**Q7: `Remove()` 中如果数量减到 0 会发生什么？**

<details>
<summary>参考答案</summary>

从列表中移除该 ItemStack。
</details>

---

**Q8: `Init()` 是如何初始化背包的？**

<details>
<summary>参考答案</summary>

清空当前列表，从 `_defaultItems` 复制。
</details>

---

**Q9: `CookRecipeEventRaised()` 做了什么？**

<details>
<summary>参考答案</summary>

1. 检查是否有配方
2. 检查是否有足够材料
3. 消耗材料
产物
5. 保存
</details>

---

**Q10: `ItemPicker.PickItem()` 是如何触发添加物品的？**

<details>
<summary>参考答案</summary>

通过事件（`ItemEventChannelSO`）通知 `InventoryManager`。
</details>

---

**Q11: `CollectableItem` 的作用是什么？**

<details>
<summary>参考答案</summary>

场景中的可收集物品，持有 ItemSO 引用，支持动画。
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`InventorySO` 和 `HealthSO` 都使用 SO 存储数据，有什么区别？**

<details>
<summary>参考答案</summary>

- `InventorySO`：集合数据（列表），支持运行时修改
- `HealthSO`：单一数据（当前/最大生命值），支持实例化
</details>

---

**Q13: 【漏步后果】如果 `Add()` 忘记检查 `ItemType.Use`，会有什么问题？**

<details>
<summary>参考答案</summary>

武器等不可堆叠物品会被错误堆叠。
</details>

---

**Q14: 【架构陷阱】如果两个系统同时修改背包，会有什么问题？**

<details>
<summary>参考答案</summary>

可能导致数据不一致。需要锁或队列机制。
</details>

---

**Q15: 【同底座差异】`ItemSO` 和 `ItemStack` 的关系是什么？**

<details>
<summary>参考答案</summary>

- `ItemSO`：物品定义（静态数据）
- `ItemStack`：物品实例（物品 + 数量）
</details>

---

**Q16: 【漏步后果】如果 `CookRecipeEventRaised()` 忘记保存，会有什么问题？**

<details>
<summary>参考答案</summary>

重启游戏后配方进度丢失。
</details>

---

**Q17: 【架构陷阱】`InventoryManager` 和 `InventorySO` 为什么要分离？**

<details>
<summary>参考答案</summary>

- `InventorySO`：纯数据
- `InventoryManager`：逻辑 + 事件处理
</details>

---

**Q18: 【漏步后果】如果 `Remove()` 中忘记检查数量 <= 0，会有什么问题？**

<details>
<summary>参考答案</summary>

背包中会有数量为 0 的物品。
</details>

---

## �️ 实操题

**Q19: 实现一个 `Shop`，允许购买物品。**

<details>
<summary>参考答案</summary>

```csharp
public class Shop : MonoBehaviour
{
    [SerializeField] private InventorySO _playerInventory;
    [SerializeField] private ItemSO[] _itemsForSale;
    
    public void BuyItem(ItemSO item, int cost)
    {
        // 检查金币
        if (_playerInventory.Count(_goldItem) >= cost)
        {
            _playerInventory.Remove(_goldItem, cost);
            _playerInventory.Add(item);
        }
    }
}
```
</details>

---

**Q20: Book`，显示可制作的配方。**

<details>
<summary>参考答案</summary>

```csharp
public class RecipeBook : MonoBehaviour
{
    [SerializeField] private InventorySO _inventory;
    [SerializeField] private ItemSO[] _recipes;
    
    public List<ItemSO> GetCraftableRecipes()
    {
        var result = new List<ItemSO>();
        foreach (var recipe in _recipes)
        {
            if (_inventory.HasIngredients(recipe.ingredients))
                result.Add(recipe);
        }
        return result;
    }
}
```
</details>

---

**Q21: 假设有一个 Bug未被消耗。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. `CookRecipeEventRaised()` 中忘记调用 `Remove()`
2. 材料检查逻辑错误
3. 事件未正确触发
</details>
