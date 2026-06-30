# Factory 考题

## � 概念题

**Q1: `IFactory<T>` 接口的作用是什么？**

<details>
<summary>参考答案</summary>

定义创建对象的契约，将对象的创建逻辑与使用方解耦。
</details>

---

**Q2: 为什么 `FactorySO<T>` 是抽象类而非接口？**

<details>
<summary>参考答案</summary>

因为需要继承 `ScriptableObject` 以支持 Inspector 配置，而 C# 不支持多继承。
</details>

---

**Q3: 工厂模式在这个项目中的主要用途是什么？**

<details>
<summary>参考答案</summary>

为对象池（Pool）提供创建新对象的能力，实现创建逻辑的解耦。
</details>

---

**Q4: `FactorySO<T>` 的 `Create()` 为什么是抽象的？**

<details>
<summary>参考答案</summary>

不同类型的对象创建方式不同（new / Instantiate / Addressables），必须由子类实现。
</details>

---

**Q5: 工厂与池的关系是什么？**

<details>
<summary>参考答案</summary>

池通过工厂创建新对象，通过自身逻辑复用对象。工厂是池的依赖注入点。
</details>

---

## 🟡 机制题

**Q6: 如果池没有配置工厂，调用 `Request()` 会发生什么？**

<details>
<summary>参考答案</summary>

`Factory` 属性为 null，调用 `Create()` 会抛出 NullReferenceException。
</details>

---

**Q7: 工厂创建的 GameObject 应该如何初始化？**

<details>
<summary>参考答案</summary>

在 `Create()` 方法中完成初始化（设置位置、激活状态等），或者在池的 `Request()` / `Create()` 中处理。
</details>

---

**Q8: 为什么使用泛型接口而非非泛型？**

<details>
<summary>参考答案</summary>

类型安全，避免装箱拆箱，编译期检查。
</details>

---

**Q9: 工厂模式与直接 `new` 相比有什么优势？**

<details>
<summary>参考答案</summary>

1. 解耦创建逻辑
2. 支持配置化
3. 便于测试（mock）
4. 支持多态
</details>

---

**Q10: 如果工厂创建的对象需要参数怎么办？**

<details>
<summary>参考答案</summary>

1. 在工厂 SO 中配置参数
2. 使用 `Create(ConfigSO config)` 重载
3. 使用 Builder 模式
</details>

---

**Q11: 工厂模式在这个项目中的局限性是什么？**

<details>
<summary>参考答案</summary>

只支持无参创建。如果需要参数化创建，需要扩展接口。
</details>

---

## � 架构陷阱题

**Q12: 【同底座差异】`FactorySO<T>` 与直接使用 `new` 有什么区别？**

<details>
<summary>参考答案</summary>

- `FactorySO<T>`：可配置、可替换、支持多态
- `new`：硬编码、不可配置
</Q13: 【漏步后果】如果工厂创建的对象没有被池回收，会有什么问题？**

<details>
<summary>参考答案</summary>

内存泄漏。对象一直存在但无法被复用。
</details>

---

**Q14: 【架构陷阱】如果两个池使用同一个工厂实例，会有什么问题？**

<details>
<summary>参考答案</summary>

如果工厂有状态（如计数器），两个池会互相干扰。应该每个池有独立的工厂实例。
</details>

---

**Q15: 【同底座差异】`IFactory<T>` 与 `Func<T>` 有什么区别？**

<details>
<summary>参考答案</summary>

- `IFactory<T>`：接口，可扩展，支持多态
- `Func<T>`：委托，轻量但功能有限
</details>

---

**Q16: 【漏步后果】如果工厂创建的对象没有正确初始化，会有什么问题？**

<details>
<summary>参考答案</summary>

对象可能处于无效状态，导致运行时错误。
</details>

---

**Q17: 【架构陷阱】工厂模式与单例模式的主要区别是什么？**

<details>
<summary>参考答案</summary>

- 工厂：关注对象创建
- 单例：关注实例唯一性
</details>

---

**Q18: 【漏步后果】如果工厂的 `Create()` 返回 null，池会如何处理？**

<details>
<summary>参考答案</summary>

池会将 null 压入栈，后续 `Request()` 返回 null，导致空引用异常。
</details>

---

## ✍️ 实操题

**Q19: 实现一个创建 `HealthSO` 的工厂。**

<details>
<summary>参考答案</summary>

```csharp
[CreateAssetMenu(menuName = "Factory/Health Factory")]
public class HealthFactorySO : FactorySO<HealthSO>
{
    [SerializeField] private int _maxHealth = 100;
    
    public override HealthSO Create()
    {
        var health = ScriptableObject.CreateInstance<HealthSO>();
        health.SetMaxHealth(_maxHealth);
        health.SetCurrentHealth(_maxHealth);
        return health;
    }
}
```
</details>

---

**Q20: 实现一个创建 GameObject 的工厂。**

<details>
<summary>参考答案</summary>

```csharp
[CreateAssetMenu(menuName = "Factory/GameObject Factory")]
public class GameObjectFactorySO : FactorySO<GameObject>
{
    [SerializeField] private GameObject _prefab;
    
    public override GameObject Create()
    {
        return Instantiate(_prefab);
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：池返回的对象状态不正确。请列出至少 2 个可能的原因。**

<details>
<summary>参考答案</summary>

1. 工厂创建时未正确初始化
2. 池的 `Request()` 未重置对象状态
3. 对象在返回前未清理状态
</details>
