# Dialogues 考题

## � 概念题

**Q1: `DialogueType` 枚举定义了哪些类型？**

<details>
<summary>参考答案</summary>

Start、Completion、Incompletion、Default。
</details>

---

**Q2: `ChoiceActionType` 枚举定义了哪些类型？**

<details>
<summary>参考答案</summary>

DoNothing、ContinueWithStep、WinningChoice、LosingChoice。
</details>

---

**Q3: `DialogueManager` 如何推进对话？**

<details>
<summary>参考答案</summary>

使用 `_counterDialogue`（段索引）和 `_counterLine`（行索引），玩家按按钮推进。
</details>

---

**Q4: 对话开始时游戏状态如何变化？**

<details>
<summary>参考答案</summary>

切换到 `GameState.Dialogue`，启用 `DialogueInput`。
</details>

---

**Q5: 对话结束时如何恢复状态？**

<details>
<summary>参考答案</summary>

调用 `ResetToPreviousGameState()` 恢复到之前的状态。
</details>

---

## 🟡 机制题

**Q6: 如果对话中有选项，会发生什么？**

<details>
<summary>参考答案</summary>

暂停推进，显示选项，等待玩家选择。
</details>

---

**Q7: `MakeDialogueChoice()` 如何处理选项？**

<details>
<summary>参考答案</summary>

根据 `ChoiceActionType` 触发相应逻辑，然后继续下一个对话或结束。
</details>

---

**Q8: 如果 `AdvanceDialogueEvent` 在选项时未取消订阅，会有什么问题？**

<details>
<summary>参考答案</summary>

玩家按按钮会同时触发推进和选择，导致逻辑混乱。
</details>

---

**Q9: `DisplayDialogueLine()` 是如何显示对话的？**

<details>
<summary>参考答案</summary>

通过 `DialogueLineChannelSO` 通知UI显示。
</details>

---

**Q10: 如果两个对话同时开始，会发生什么？**

<details>
<summary>参考答案</summary>

后开始的覆盖先开始的，可能导致状态不一致。
</details>

---

**Q11: `DialogueDataSO` 的 `FinishDialogue()` 做了什么？**

<details>
<summary>参考答案</summary>

触发 `EndOfDialogueEvent` 事件。
</details>

---

## 🔴 **Q12: 【同底座差异】`DialogueManager` 和 `QuestManagerSO` 都管理流程，有什么区别？**

<details>
<summary>参考答案</summary>

- `DialogueManager`：对话流程，UI驱动
- `QuestManagerSO`：任务流程，数据驱动
</details>

---

**Q13: 【漏步后果】如果忘记在选项时取消 `AdvanceDialogueEvent` 订阅，会有什么问题？**

<details>
<summary>参考答案</summary>

玩家按按钮会触发推进，跳过选项。
</details>

---

**Q14: 【架构陷阱】如果对话结束时忘记恢复游戏状态，会有什么问题？**

<details>
<summary>参考答案</summary>

游戏会卡在 `Dialogue` 状态，玩家无法操作。
</details>

---

**Q15: 【同底座差异】`DialogueDataSO` 和 `ItemSO` 都使用 SO 存储数据，有什么区别？**

<details>
<summary>参考答案</summary>

- `DialogueDataSO`：对话数据（行 + 选项）
- `ItemSO`：物品数据（名称 + 图标 + 类型）
</details>

---

**Q16: 【漏步后果】如果 `DisplayDialogueData()` 忘记重置索引，会有什么问题？**

<details>
<summary>参考答案</summary>

对话可能从错误的位置开始。
</details>

---

**Q17: 【架构陷阱】对话系统与任务系统的主要耦合点是什么？**

<details>
<summary>参考答案</summary>

选项的 `ContinueWithStep` 类型触发任务推进。
</details>

---

**Q18: 【漏步后果】如果 `OnAdvance()` 中忘记检查行结束，会有什么问题？**

<details>
<summary>参考答案</summary>

索引越界异常。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `DialogueUI`，显示对话和选项。**

<details>
<summary>参考答案</summary>

```csharp
public class DialogueUI : MonoBehaviour
{
    [SerializeField] private Text _dialogueText;
    [SerializeField] private Image _portraitImage;
    [SerializeField] private GameObject _choicesPanel;
    [SerializeField] private Button _choiceButtonPrefab;
    
    private DialogueManager _dialogueManager;
    
    private void OnEnable()
    {
        _dialogueManager.OnLineDisplayed += DisplayLine;
        _dialogueManager.OnChoicesDisplayed += DisplayChoices;
    }
    
    private void OnDisable()
    {
        _dialogueManager.OnLineDisplayed -= DisplayLine;
        _dialogueManager.OnChoicesDisplayed -= DisplayChoices;
    }
    
    private void DisplayLine(string text, ActorSO actor)
    {
        _dialogueText.text = text;
        _portraitImage.sprite = actor.portrait;
        _choicesPanel.SetActive(false);
    }
    
    private void DisplayChoices(List<Choice> choices)
    {
        _choicesPanel.SetActive(true);
        // 动态创建选项按钮
    }
}
```
</details>

---

**Q20: 实现一个 `DialogueTrigger`，触发对话。**

<details>
<summary>参考答案</summary>

```csharp
public class DialogueTrigger : MonoBehaviour
{
    [SerializeField] private DialogueDataSO _dialogue;
    [SerializeField] private DialogueManager _dialogueManager;
    
    public void Trigger()
    {
        _dialogueManager.StartDialogue(_dialogue);
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：对话结束后无法恢复游戏。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. `ResetToPreviousGameState()` 未被调用
2. `_previousGameState` 未正确设置
3. 状态切换时广播了错误的事件
</details>
