# Events 考题

## 🟢 概念题

**Q1: 为什么事件通道使用 ScriptableObject 而非静态类或单例？**

<details>
<summary>参考答案</summary>

1. **可配置性**：在 Inspector 中配置引用，无需代码耦合
2. **跨场景**：SO 是资产，不绑定到特定场景
3. **可替换**：不同场景/模式可以使用不同的 Channel 实例
4. **可测试**：可以 mock Channel 进行单元测试
</details>

---

**Q2: `VoidEventChannelSO` 使用 `UnityAction` 而非 `event UnityAction`，有什么区别？**

<details>
<summary>参考答案</summary>

- `UnityAction OnEventRaised`：外部可以直接调用 `Invoke()`（危险），也可以清空所有订阅
- `event UnityAction OnEventRaised`：外部只能 `+=` / `-=`，不能直接 Invoke 或清空
- 原项目混合使用：Void 用 `UnityAction`（允许外部 Invoke），其他用 `event`（限制外部操作）
</details>

---

**Q3: `VoidEventListener` 的作用是什么？为什么不能直接在 Inspector 中订阅 Channel？**

<details>
<summary>参考答案</summary>

`VoidEventListener` 是 SO 事件到 UnityEvent 的桥接器。因为：
1. Channel 的 `OnEventRaised` 是 C# 事件，Inspector 无法直接配置
2. UnityEvent 可以在 Inspector 中配置响应方法
3. 桥接器在 `OnEnable`/`OnDisable` 中管理订阅生命周期
</details>

---

**Q4: 如果两个 MonoBehaviour 订阅同一个 Channel，事件触发时执行顺序是什么？**

<details>
<summary>参考答案</summary>

按订阅顺序执行（即 `+=` 的顺序）。先订阅的先执行。
</details>

---

**Q5: `RaiseEvent()` 中为什么要检查 `OnEventRaised != null`？**

<details>
<summary>参考答案</summary>

如果没有订阅者，`OnEventRaised` 为 null，直接调用会抛出 NullReferenceException。`?.Invoke()` 或 null 检查可以避免此问题。
</details>

---

## 🟡 机制题

**Q6: 如果 MonoBehaviour 在 `OnEnable` 中订阅，但在 `OnDisable` 中忘记取消订阅，会发生什么？**

<details>
<summary>参考答案</summary>

1. **内存泄漏**：委托持有对象的引用，阻止 GC 回收
2. **空引用异常**：如果对象已被销毁，调用委托会失败
3. **重复执行**：如果对象重新启用，会再次订阅，导致重复响应
</details>

---

**Q7: 场景切换时，SO Channel 的订阅者列表会丢失吗？**

<details>
<summary>参考答案</summary>

不会。SO 是资产，不随场景加载/卸载。但订阅的 MonoBehaviour 如果随场景销毁，委托引用会变成"悬空"。正确做法是在 `OnDisable` 中取消订阅。
</details>

---

**Q8: 如果 `RaiseEvent()` 内部抛出一个异常，后续订阅者还会执行吗？**

<details>
<summary>参考答案</summary>

不会。委托链中某个订阅者抛出异常会中断后续执行。需要手动遍历委托列表并 try-catch 每个调用。
</details>

---

**Q9: 为什么 `BoolEventChannelSO` 使用 `event` 关键字，而 `VoidEventChannelSO` 不使用？**

<details>
<summary>参考答案</summary>

这是一个设计选择（可能不一致）。使用 `event` 可以防止外部代码清空所有订阅（ised = null`）或直接调用 `Invoke()`。
</details>

---

**Q10: 如何在事件传递时携带多个参数？**

<details>
<summary>参考答案</summary>

1. 创建自定义结构体/类作为参数
2. 使用 `EventChannelSO<CustomArgs>`
3. 或者使用多个 Channel 分别传递
</details>

---

**Q11: `VoidEventListener` 的 `Respond()` 方法为什么是 private？**

<details>
<summary>参考答案</summary>

封装性。`Respond()` 只应由 Channel 事件触发，不应被外部直接调用。private 限制访问。
</details>

---

## � 架构陷阱题

**Q12: 【同底座差异】`VoidEventChannelSO` 和 `BoolEventChannelSO` 的实现有什么微妙差异？**

<details>
<summary>参考答案</summary>

- `VoidEventChannelSO`：`public UnityAction OnEventRaised`（无 event 关键字）
- `BoolEventChannelSO`：`public event UnityAction<bool> OnEventRaised`（有 event 关键字）
- 差异：前者外部可以 `Invoke()` 和赋值，后者只能 `+=` / `-=`
</details>

---

**Q13: 【漏步后果】如果在 `OnEnable` 中订阅，但在 `OnDestroy` 中取消订阅（而非 `OnDisable`），会有什么问题？**

<details>
<summary>参考答案</summary>

如果对象被禁用（`SetActive(false)`）而非销毁，`OnDisable` 会触发但 `OnDestroy` 不会。此时：
1. 对象禁用时仍持有订阅
2. 如果 Channel 在对象禁用时触发事件，会尝试调用已禁用对象的方法（Unity 允许但可能产生意外行为）
</details>

---

**Q14: 【架构陷阱】如果两个不同的 Channel SO 实例被错误地配置为同一个，会发生什么？**

<details>
<summary>参考答案</summary>

所有订阅者会收到两个 Channel 的事件。例如：`_onPlayerDeath` 和 `_onGameOver` 指向同一个 SO，触发任一事件都会导致两个响应都执行。
</details>

---

**Q15: 【同底座差异】`VoidEventListener` 和直接在代码中订阅 Channel 有什么区别？**

<details>
<summary>参考答案</summary>

- `VoidEventListener`：通过 UnityEvent 在 Inspector 中配置响应，适合设计师配置
- 代码订阅：在 `OnEnable`/`OnDisable` 中手动 `+=` / `-=`，适合程序员控制
- 关键区别：Inspector 配置更灵活，但代码订阅更明确
</details>

---

**Q16: 【漏步后果】如果 `RaiseEvent()` 在场景加载完成前调用，会有什么问题？**

<details>
<summary>参考答案</summary>

订阅者可能尚未执行 `OnEnable`，导致事件丢失。解决方案：
1. 使用协程延迟到EndOfFrame
2. 确保订阅者在发布者之前初始化
3. 使用延迟初始化模式
</details>

---

**Q17: 【架构陷阱】事件系统与直接方法调用相比，主要缺点是什么？**

<details>
<summary>参考答案</summary>

1. **调试困难**：调用链不明显，难以追踪事件流
2. **性能开销**：委托调用比直接方法调用略慢
3. **内存分配**：委托链合并时可能产生堆分配
4. **顺序依赖**：执行顺序依赖订阅顺序，可能产生隐式耦合
</details>

---

**Q18: 【漏步后果】如果在 `Update` 中每帧调用 `RaiseEvent()`，会有什么性能影响？**

<details>
<summary>参考答案</summary>

1. 每帧遍历委托链，即使无订阅者也有 null 检查开销
2. 如果有大量订阅者，每帧调用会产生 CPU 开销
3. 如果订阅者执行复杂逻辑，会放大性能问题
4. 解决方案：只在状态变化时触发，而非每帧
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `EventChannelSO` 的扩展方法，允许一次性订阅（事件触发后自动取消订阅）。**

<details>
<summary>参考答案</summary>

```csharp
public static class EventExtensions
{
    public static void SubscribeOnce<T>(this EventChannelSO<T> channel, Action<T> handler)
    {
        Action<T> wrapper = null;
        wrapper = (value) =>
        {
            handler(value);
            channel.OnEventRaised -= wrapper;
        };
        channel.OnEventRaised += wrapper;
    }
}
```
</details>

---

**Q20: 实现一个事件聚合器，允许多个事件触发同一个响应。**

<details>
<summary>参考答案</summary>

```csharp
public class EventAggregator : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO[] _channels;
    private UnityEvent _response;

    private void OnEnable()
    {
        foreach (var channel in _channels)
            channel.OnEventRaised += OnAnyEvent;
    }

    private void OnDisable()
    {
        foreach (var channel in _channels)
            channel.OnEventRaised -= OnAnyEvent;
    }

    private void OnAnyEvent() => _response.Invoke();
}
```
</details>

---

**Q21: 假设有一个 Bug：事件触发了但响应没有执行。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. **订阅者未启用**：MonoBehaviour 被禁用，`OnEnable` 未执行
2. **订阅已取消**：`OnDisable` 中取消了订阅但发布者不知道
3. **Channel 引用错误**：发布者和订阅者使用了不同的 Channel SO 实例
4. **异常中断**：前一个订阅者抛出异常，后续订阅者未执行
5. **时序问题**：事件在订阅前触发
</details>
