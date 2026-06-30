# RuntimeAnchors 考题

## � 概念题

**Q1: `RuntimeAnchorBase<T>` 的作用是什么？**

<details>
<summary>参考答案</summary>

在运行时存储跨场景的 Unity 对象引用，利用 SO 的资产特性实现持久化。
</details>

---

**Q2: 为什么使用 SO 而非静态类存储运行时引用？**

<details>
<summary>参考答案</summary>

1. 可配置性：在 Inspector 中配置引用
2. 可替换：不同场景/模式可以使用不同的锚点实例
3. 可测试：可以 mock 锚点进行单元测试
</details>

---

**Q3: `isSet` 属性的作用是什么？**

<details>
<summary>参考答案</summary>

允许消费者检查锚点是否已设置，避免访问未初始化的 Value。
</details>

---

**Q4: `OnDisable()` 中为什么要调用 `Unset()`？**

<details>
<summary>参考答案</summary>

防止悬空引用。SO 禁用时清除存储的引用，避免消费者访问已销毁的对象。
</details>

---

**Q5: `Provide()` 中为什么要检查 null？**

<details>
<summary>参考答案</summary>

避免存储无效引用。如果允许 null，消费者可能误以为锚点已设置。
</details>

---

## � 机制题

**Q6: 如果两个系统同时调用 `Provide()`，会发生什么？**

<details>
<summary>参考答案</summary>

后调用的覆盖先调用的。`OnAnchorProvided` 事件会触发两次。
</details>

---

**Q7: 如果消费者在 `OnEnable` 中检查 `isSet` 为 false，之后锚点被设置，消费者会收到通知吗？**

<details>
<summary>参考答案</summary>

会，如果消费者订阅了 `OnAnchorProvided` 事件。
</details>

---

**Q8: `TransformAnchor` 是如何实现类型特化的？**

<details>
<summary>参考答案</summary>

通过继承 `RuntimeAnchorBase<Transform>`，无需额外代码。
</details>

---

**Q9: 如果锚点被 `Unset()` 后消费者访问 `Value`，会返回什么？**

<details>
<summary>参考答案</summary>

返回 null（因为 `_isSet` 为 false）。
</details>

---

**Q10: `OnAnchorProvided` 事件是什么时候触发的？**

<details>
<summary>参考答案</summary>

在 `Provide()` 方法中，设置完 `_value` 和 `_isSet` 后触发。
</details>

---

**Q11: 如果消费者在 `OnDisable` 中忘记取消订阅 `OnAnchorProvided`，会有什么问题？**

<details>
<summary>参考答案</summary>

内存泄漏。事件持有消费者引用，阻止 GC 回收。
</details>

---

## � 架构陷阱题

**Q12: 【同底座差异】`RuntimeAnchorBase<T>` 和直接传递引用有什么区别？**

<details>
<summary>参考答案</直接传递：紧耦合，场景切换时引用丢失
- 锚点：解耦，跨场景持久化
</details>

---

**Q13: 【漏步后果】如果消费者未检查 `isSet` 直接访问 `Value`，会有什么问题？**

<details>
<summary>参考答案</summary>

可能访问到 null 或过时的引用，导致空引用异常或逻辑错误。
</details>

---

**Q14: 【架构陷阱】如果两个生产者使用同一个锚点，会有什么问题？**

<details>
<summary>参考答案</summary>

后生产者的值覆盖前生产者的值，可能导致消费者使用错误的引用。
</details>

---

**Q15: 【同底座差异】`Provide()` 和直接赋值有什么区别？**

<details>
<summary>参考答案</summary>

- `Provide()`：包含 null 检查、设置 isSet、触发事件
- 直接赋值：无保护，可能导致无效状态
</details>

---

**Q16: 【漏步后果】如果 `OnDisable` 中忘记调用 `Unset()`，会有什么问题？**

<details>
<summary>参考答案</summary>

悬空引用。消费者可能访问已销毁的对象。
</details>

---

**Q17: 【架构陷阱】锚点与事件系统的主要区别是什么？**

<details>
<summary>参考答案</summary>

- 锚点：存储状态（引用），消费者主动查询
- 事件：推送通知，消费者被动响应
</details>

---

**Q18: 【漏步后果】如果消费者在 `OnEnable` 中未订阅 `OnAnchorProvided`，会有什么问题？**

<details>
<summary>参考答案</summary>

如果锚点之后被设置，消费者不会收到通知，可能错过初始化时机。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `GameObjectAnchor`，用于存储任意 GameObject 引用。**

<details>
<summary>参考答案</summary>

```csharp
[CreateAssetMenu(menuName = "Runtime Anchors/GameObject")]
public class GameObjectAnchor : RuntimeAnchorBase<GameObject> { }
```
</details>

---

**Q20: 实现一个消费者，在锚点设置后执行初始化。**

<details>
<summary>参考答案</summary>

```csharp
public class Consumer : MonoBehaviour
{
    [SerializeField] private TransformAnchor _anchor;
    
    private void OnEnable()
    {
        _anchor.OnAnchorProvided += OnAnchorSet;
        if (_anchor.isSet) OnAnchorSet();
    }
    
    private void OnDisable() => _anchor.OnAnchorProvided -= OnAnchorSet;
    
    private void OnAnchorSet()
    {
        // 使用 _anchor.Value 进行初始化
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：消费者在锚点设置后仍使用旧值。请列出至少 2 个可能的原因。**

<details>
<summary>参考答案</summary>

1. 消费者未订阅 `OnAnchorProvided` 事件
2. 消费者在事件处理中使用了缓存的引用
3. 锚点被 `Unset()` 后未重新设置
</details>
