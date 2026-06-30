# Input 考题

## � 概念题

**Q1: `InputReader` 为什么是 MonoBehaviour 而非纯 SO？**

<details>
<summary>参考答案</summary>

因为需要绑定 InputSystem 的回调，MonoBehaviour 可以访问 `OnEnable`/`OnDisable` 生命周期。
</details>

---

**Q2: 输入模式切换的作用是什么？**

<details>
<summary>参考答案</summary>

不同游戏状态需要不同的输入响应。例如对话时移动无效，菜单时攻击无效。
</details>

---

**Q3: `OnInteract()` 中为什么要检查 `GameState.Gameplay`？**

<details>
<summary>参考答案</summary>

防止在非游戏状态下触发交互（如对话时拾取物品）。
</details>

---

**Q4: `GameInput` 类是如何生成的？**

<details>
<summary>参考答案</summary>

通过 Unity Input System 的 `InputActionAsset` 代码生成功能自动生成。
</details>

---

**Q5: `InputReader` 的事件为什么初始化为 `delegate { }`？**

<details>
<summary>参考答案</summary>

避免空检查。可以直接调用 `JumpEvent.Invoke()` 而无需检查 null。
</details>

---

## 🟡 机制题

**Q6: 如果两个系统订阅同一个输入事件，会发生什么？**

<details>
<summary>参考答案</summary>

两个系统都会响应。可能导致冲突（如同时打开菜单和拾取物品）。
</details>

---

**Q7: `EnableGameplayInput()` 做了什么？**

<details>
<summary>参考答案</summary>

启用 Gameplay Action Map，禁用其他。
</details>

---

**Q8: 如果输入模式未正确切换，会有什么问题？**

<details>
<summary>参考答案</summary>

玩家可能在对话时移动，或在菜单时攻击。
</details>

---

**Q9: `OnMove()` 是如何传递输入值的？**

<details>
<summary>参考答案</summary>

通过 `ctx.ReadValue<Vector2>()` 读取摇杆/键盘输入，触发 `MoveEvent`。
</details>

---

**Q10: 如果 `OnDisable` 中忘记禁用输入，会有什么问题？**

<details>
<summary>参考答案</summary>

输入可能在对象禁用后仍然响应。
</details>

---

**Q11: `DisableAllInput()` 的作用是什么？**

<details>
<summary>参考答案</summary>

禁用所有 Action Map，用于暂停游戏或场景切换。
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`InputReader` 和 `EventChannelSO` 都使用事件，有什么区别？**

<details>
<summary>参考答案</summary>

- `InputReader`：输入事件，高频，需要状态过滤
- `EventChannelSO`：业务事件，低频，持久化
</details>

---

**Q13: 【漏步后果】如果忘记在 `OnDisable` 中取消订阅输入事件，会有什么问题？**

<details>
<summary>参考答案</summary>

内存泄漏，已销毁对象仍响应输入。
</details>

---

**Q14: 【架构陷阱】如果输入事件在高频回调中执行复杂逻辑，会有什么问题？**

<details>
<summary>参考答案</summary>

性能问题。输入回调应只做最小工作，复杂逻辑应延迟到 `Update`。
</details>

---

**Q15: 【同底座差异】`InputReader` 和 `VoidEventListener` 都桥接事件，有什么区别？**

<details>
<summary>参考答案</summary>

- `InputReader`：InputSystem → C# 事件
- `VoidEventListener`：SO 事件 → UnityEvent
</details>

---

**Q16: 【漏步后果】如果 `OnInteract()` 忘记检查 `GameState`，会有什么问题？**

<details>
<summary>参考答案</summary>

玩家可能在对话时拾取物品，导致状态混乱。
</details>

---

**Q17: 【架构陷阱】为什么不在每个系统中直接读取输入？**

<details>
<summary>参考答案</summary>

解耦。输入逻辑集中管理，便于切换输入设备、支持重绑定。
</details>

---

**Q18: 【漏步后果】如果 `EnableGameplayInput()` 忘记禁用其他 Action Map，会有什么问题？**

<details>
<summary>参考答案</summary>

多个输入模式同时激活，导致冲突。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `SprintAction`，按住Shift时加速移动。**

<details>
<summary>参考答案</summary>

```csharp
public class SprintAction : StateAction
{
    private InputReader _input;
    
    public override void Awake(StateMachine sm)
    {
        _input = sm.GetComponent<InputReader>();
        _input.StartedRunning += OnSprintStart;
        _input.StoppedRunning += OnSprintStop;
    }
    
    private void OnSprintStart() => isSprinting = true;
    private void OnSprintStop() => isSprinting = false;
    
    public override void OnUpdate()
    {
        speed = isSprinting ? runSpeed : walkSpeed;
    }
}
```
</details>

---

**Q20: 实现一个 `InputBuffer`，缓存输入防止丢失。**

<details>
<summary>参考答案</summary>

```csharp
public class InputBuffer
{
    private float _bufferTime = 0.2f;
    private float _lastJump    public void BufferJump() => _lastJumpTime = Time.time;
    
    public bool ConsumeJump()
    {
        if (Time.time - _lastJumpTime <= _bufferTime)
        {
            _lastJumpTime = 0;
            return true;
        }
        return false;
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：输入无响应。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. Action Map 未启用
2. 回调未绑定
3. 状态过滤阻止了输入
</details>
