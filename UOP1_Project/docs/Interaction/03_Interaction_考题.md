# Interaction 考题

## 🟢 概念题

**Q1: `InteractionManager` 为什么使用 `LinkedList` 而非 `List`？**

<details>
<summary>参考答案</summary>

`LinkedList` 支持 O(1) 删除任意节点，适合频繁添加/移除交互对象。
</details>

---

**Q2: `InteractionType` 枚举定义了哪些类型？**

<details>
<summary>参考答案</summary>

None、PickUp、Cook、Talk。
</details>

---

**Q3: 交互优先级是如何确定的？**

<details>
<summary>参考答案</summary>

`LinkedList` 的第一个元素是最高优先级，按进入触发器顺序添加。
</details>

---

**Q4: `OnTriggerChangeDetected()` 是如何被调用的？**

<details>
<summary>参考答案</summary>

由 `ZoneTriggerController` 在 `OnTriggerEnter`/`OnTriggerExit` 中调用。
</details>

---

**Q5: 拾取物品时为什么不立即销毁对象？**

<details>
<summary>参考答案</summary>

等待动画播放到拾取帧，由动画事件调用 `Collect()`。
</details>

---

## 🟡 机制题

**Q6: 如果玩家同时站在两个可拾取物品上，会拾取哪个？**

<details>
<summary>参考答案</summary>

先进入触发器的（`LinkedList.First`）。
</details>

---

**Q7: `Collect()` 中为什么要先 `RemoveFirst()` 再销毁对象？**

<details>
<summary>参考答案</summary>

防止对象销毁后仍在链表中，导致后续操作错误。
</details>

---

**Q8: 如果 `RemoveInteraction()` 中忘记 `break`，会有什么问题？**

<details>
<summary>参考答案</summary>

继续遍历已移除的节点，可能导致异常。
</details>

---

**Q9: `UpdateUI()` 是在什么时候调用的？**

<details>
<summary>参考答案</summary>

添加/移除交互后，以及拾取后。
</details>

---

**Q10: 如果玩家在交互动画期间离开触发器，会发生什么？**

<details>
<summary>参考答案</summary>

交互对象被移除，动画结束时 `Collect()` 可能找不到对象。
</details>

---

**Q11: `InteractionUIEventChannelSO` 的作用是什么？**

<details>
<summary>参考答案</summary>

通知UI显示/隐藏交互提示。
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`InteractionManager` 和 `QuestManagerSO` 都管理交互，有什么区别？**

<details>
<summary>参考答案</summary>

- `InteractionManager`：物理交互（拾取、对话、烹饪）
- `QuestManagerSO`：任务逻辑交互（步骤推进）
</details>

---

**Q13: 【漏步后果】如果忘记在 `OnDisable` 中取消订阅 `InteractEvent`，会有什么问题？**

<details>
<summary>参考答案</summary>

内存泄漏，已销毁对象仍响应交互输入。
</details>

---

**Q14: 【架构陷阱】如果两个交互对象有相同优先级，如何处理？**

<details>
<summary>参考答案</summary>

按进入顺序，先进入的优先。
</details>

---

**Q15: 【同底座差异】`Interaction` 类和 `ItemStack` 都包含对象引用，有什么区别？**

<details>
<summary>参考答案</summary>

- `Interaction`：运行时交互目标（GameObject + 类型）
- `ItemStack`：背包数据（ItemSO + 数量）
</details>

---

**Q16: 【漏步后果】如果 `AddInteraction()` 忘记调用 `UpdateUI()`，会有什么问题？**

<details>
<summary>参考答案</summary>

UI不会显示交互提示，玩家不知道可以交互。
</details>

---

**Q17: 【架构陷阱】为什么不在每个交互对象上独立处理输入？**

<details>
<summary>参考答案</summary>

集中管理优先级，避免多个对象同时响应输入。
</details>

---

**Q18: 【漏步后果】如果 `Collect()` 中忘记销毁对象，会有什么问题？**

<details>
<summary>参考答案</summary>

物品对象残留在场景中，可能被重复拾取。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `InteractionPrompt`，显示交互按键提示。**

<details>
<summary>参考答案</summary>

```csharp
public class InteractionPrompt : MonoBehaviour
{
    [SerializeField] private TMPro.TextMeshProUGUI _text;
    [SerializeField] private GameObject _panel;
    
    public void Show(string key, string action)
    {
        _text.text = $"按 {key} {action}";
        _panel.SetActive(true);
    }
    
    public void Hide() => _panel.SetActive(false);
}
```
</details>

---

**Q20: 实现一个 `CookingStation`，支持烹饪交互。**

<details>
<summary>参考答案</summary>

```csharp
public class CookingStation : MonoBehaviour
{
    [SerializeField] private InteractionManager _interactionManager;
    
    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player"))
        {
            _interactionManager.AddInteraction(InteractionType.Cook, gameObject);
        }
    }
    
    private void OnTriggerExit(Collider other)
    {
        if (other.CompareTag("Player"))
        {
            _interactionManager.RemoveInteraction(gameObject);
        }
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：交互提示不显示。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. 标签设置错误
2. `UpdateUI()` 未被调用
3. UI面板未激活
</details>