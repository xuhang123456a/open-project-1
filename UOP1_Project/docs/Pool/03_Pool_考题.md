# Pool 考题

## 🟢 概念题

**Q1: `IPool<T>` 接口定义了哪些方法？**

<details>
<summary>参考答案</summary>

`Prewarm(int)`、`Request()`、`Return(T)`。
</details>

---

**Q2: 为什么使用 `Stack<T>` 而非 `List<T>` 或 `Queue<T>`？**

<details>
<summary>参考答案</summary>

Stack 提供 LIFO 语义，最近返回的对象最先被复用，缓存局部性更好。
</details>

---

**Q3: `Prewarm()` 为什么只能调用一次？**

<details>
<summary>参考答案</summary>

通过 `HasBeenPrewarmed` 标志防止重复预热，避免创建多余对象。
</details>

---

**Q4: `ComponentPoolSO<T>` 的 `PoolRoot` 是什么？**

<details>
<summary>参考答案</summary>

一个自动创建的 GameObject，作为池中所有 Component 的父物体，保持场景整洁。
</details>

---

**Q5: `OnDisable()` 中为什么要清空栈？**

<details>
<summary>参考答案</summary>

防止内存泄漏。池禁用时，所有引用被释放，等待 GC 回收。
</details>

---

## 🟡 机制题

**Q6: 如果 `Request()` 时栈为空，会发生什么？**

<details>
<summary>参考答案</summary>

调用 `Factory.Create()` 创建新对象并返回。
</details>

---

**Q7: `ComponentPoolSO<T>.Return(T)` 做了什么？**

<details>
<summary>参考答案</summary>

1. 将对象设置为 PoolRoot 的子物体
2. 禁用 GameObject
3. 调用 `base.Return(member)` 压入栈
</details>

---

**Q8: 如果 `Factory` 未设置，调用 `Request()` 会怎样？**

<details>
<summary>参考答案</summary>

栈空时调用 `Create()` → `Factory.Create()` → NullReferenceException。
</details>

---

**Q9: `SetParent(Transform)` 的作用是什么？**

<details>
<summary>参考答案</summary>

设置池根的父物体，可以控制池的持久化行为（如 DontDestroyOnLoad）。
</details>

---

**Q10: 为什么 `PoolSO<T>` 是抽象类？**

<details>
<summary>参考答案</summary>

因为 `Factory` 属性是抽象的，需要子类提供具体实现。
</details>

---

**Q11: `OnDisable()` 中为什么要重置 `HasBeenPrewarmed`？**

<details>
<summary>参考答案</summary>

允许池重新启用后再次预热。
</details>

---

## � 架构陷阱题

**Q12: 【同底座差异】`PoolSO<T>` 和 `ComponentPoolSO<T>` 的主要区别是什么？**

<details>
<summary>参考答案</summary>

- `PoolSO<T>`：通用池，只管理对象引用
- `ComponentPoolSO<T>`：Component 专用，额外管理 GameObject 的激活状态和父子关系
</details>

---

**Q13: 【漏步后果】如果 `Return()` 时忘记禁用 GameObject，会有什么问题？**

<details>
<summary>参考答案</summary>

对象仍在场景中活跃，可能产生意外的物理碰撞、动画播放等。
</details>

---

**Q14: 【架构陷阱】如果两个系统共享同一个池，会有什么问题？**

<details>
<summary>参考答案</summary>

1. 一个系统 Return 的对象可能被另一个系统 Request
2. 对象状态可能不兼容
3. 预热数量难以平衡
</details>

---

**Q15: 【同底座差异】`Prewarm()` 和 `Create()` 有什么区别？**

<details>
<summary>参考答案</summary>

- `Prewarm(int)`：批量创建并压入栈
- `Create()`：创建单个对象（由 Factory 执行）
</details>

---

**Q16: 【漏步后果】如果 `OnDisable()` 中忘记清空栈，会有什么问题？**

<details>
<summary>参考答案</summary>

栈中的对象引用阻止 GC 回收，导致内存泄漏。
</details>

---

**Q17: 【架构陷阱】对象池与对象池 + Factory 的主要区别是什么？**

<details>
<summary>参考答案</summary>

- 纯池：创建逻辑硬编码
- 池 + Factory：创建逻辑可配置、可替换
</details>

---

**Q18: 【漏步后果】如果 `ComponentPoolSO` 的 `PoolRoot` 被外部销毁，会有什么问题？**

<details>
<summary>参考答案</summary>

后续 `Create()` 会重新创建池根，但之前 Return 的对象会丢失引用。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `BulletPool`，预创建 20 个子弹。**

<details>
<summary>参考答案</summary>

```csharp
[CreateAssetMenu(menuName = "Pool/Bullet Pool")]
public class BulletPool : ComponentPoolSO<Bullet>
{
    [SerializeField] private BulletFactorySO _factory;
    public override IFactory<Bullet> Factory => _factory;
}

// 使用
public class Weapon : MonoBehaviour
{
    [SerializeField] private BulletPool _pool;
    
    private void Start() => _pool.Prewarm(20);
    
    public void Shoot()
    {
        var bullet = _pool.Request();
        bullet.transform.position = transform.position;
    }
}
```
</details>

---

**Q20: 实现一个 `ReturnAfterDelay` 协程，自动将对象返回池中。**

<details>
<summary>参考答案</summary>

```csharp
public class Bullet : MonoBehaviour
{
    [SerializeField] private float _lifetime = 3f;
    private IPool<Bullet> _pool;
    
    public void Initialize(IPool<Bullet> pool) => _pool = pool;
    
    private void OnEnable() => StartCoroutine(AutoReturn());
    
    private IEnumerator AutoReturn()
    {
        yield return new WaitForSeconds(_lifetime);
        _pool.Return(this);
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：池返回的对象状态不正确。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. 对象在 Return 前未重置状态（如速度、动画）
2. 外部系统持有对象引用并修改了状态
3. 对象在池中被意外修改（如静态事件回调）
</details>
