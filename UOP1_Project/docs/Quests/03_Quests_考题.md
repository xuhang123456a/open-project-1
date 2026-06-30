# Quests 考题

## � 概念题

**Q1: 任务的层级结构是什么？**

<details>
<summary>参考答案</summary>

Questline → Quest → Step。一个任务线包含多个任务，一个任务包含多个步骤。
</details>

---

**Q2: `StepType` 枚举定义了哪些类型？**

<details>
<summary>参考答案</summary>

Dialogue、GiveItem、CheckItem。
</details>

---

**Q3: `QuestManagerSO` 如何追踪当前任务状态？**

<details>
<summary>参考答案</summary>

维护 `_currentQuestline`、`_currentQuest`、`_currentStep` 三个引用。
</details>

---

**Q4: `StepController` 的作用是什么？**

<details>
<summary>参考答案</summary>

上，处理交互，触发对话，推进任务。
</details>

---

**Q5: 步骤完成后会发生什么？**

<details>
<summary>参考答案</summary>

标记完成，给予奖励，检查任务是否完成，推进到下一步/任务/任务线。
</details>

---

## 🟡 机制题

**Q6: 如果当前任务的所有步骤都完成了，会发生什么？**

<details>
<summary>参考答案</summary>

任务标记完成，检查任务线是否完成，推进到下一个任务。
</details>

---

**Q7: `CheckStepValidity()` 是如何判断完成的？**

<details>
<summary>参考答案</summary>

根据StepType` 检查：Dialogue 直接通过，CheckItem 检查背包，GiveItem 直接通过。
</details>

---

**Q8: 如果玩家与没有任务的NPC交互，会发生什么？**

<details>
<summary>参考答案</summary>

播放默认对话。
</details>

---

**Q9: `QuestlineSO.OnCompleted` 事件是什么时候触发的？**

<details>
<summary>参考答案</summary>

当任务线中所有任务都完成时。
</details>

---

**Q10: 如果两个任务线都有未完成的任务，会先执行哪个？**

<details>
<summary>参考答案</summary>

按列表顺序，先执行第一个未完成的任务线。
</details>

---

**Q11: `StepSO` 的 `actorId` 有什么作用？**

<details>
<summary>参考答案</summary>

标识步骤关联的NPC，用于匹配交互对象。
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`QuestManagerSO` 和 StateMachine 都管理状态，有什么区别？**

<details>
<summary>参考答案</summary>

- `QuestManagerSO`：任务进度状态，持久化
- StateMachine：实体行为状态，运行时
</details>

---

**Q13: 【漏步后果】如果忘记检查任务是否完成，会有什么问题？**

<details>
<summary>参考答案</summary>

任务会卡在已完成的步骤上。
</details>

---

**Q14: 【架构陷阱】如果两个NPC有相同的 `actorId`，会有什么问题？**

<details>
<summary>参考答案</summary>

任务可能关联到错误的NPC。
</details>

---

**Q15: 【同底座差异】`QuestSO` 和 `StepSO` 都使用 SO 存储数据，有什么区别？**

<details>
<summary>参考答案</summary>

- `QuestSO`：任务数据（步骤列表）
- `StepSO`：步骤数据（类型、物品、对话）
</details>

---

**Q16: 【漏步后果】如果 `CompleteStep()` 忘记给予奖励，会有什么问题？**

<details>
<summary>参考答案</summary>

玩家无法获得任务奖励。
</details>

---

**Q17: 【架构陷阱】任务系统与对话系统的主要耦合点是什么？**

<details>
<summary>参考答案StepController` 触发对话，对话结果决定任务推进。
</details>

---

**Q18: 【漏步后果】如果 `StartQuestline()` 忘记检查 `isDone`，会有什么问题？**

<details>
<summary>参考答案</summary>

已完成的任务线会重新开始。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `QuestLogUI`，显示当前任务进度。**

<details>
<summary>参考答案</summary>

```csharp
public class QuestLogUI : MonoBehaviour
{
    [SerializeField] private QuestManager _questManager;
    [SerializeField] private Text _questText;
    
    private void Update()
    {
        if (_questManager.currentStep != null)
        {
            _questText.text = _questManager.currentStep.description;
        }
    }
}
```
</details>

---

**Q20: 实现一个 `ItemDeliveryStep`，要求玩家交付特定物品。**

<details>
<summary>参考答案</summary>

```csharp
[CreateAssetMenu(menuName = "Quests/Item Delivery Step")]
public class ItemDeliveryStepSO : StepSO
{
    public ItemSO deliveryItem;
    public int requiredAmount;
    
    public override bool IsComplete(InventorySO inventory)
    {
        return inventory.Count(deliveryItem) >= requiredAmount;
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：任务完成后无法开始新任务。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. `StartQuestline()` 未正确查找未完成的任务线
2. `isDone` 标志未正确设置
3. 任务列表为空
</details>
