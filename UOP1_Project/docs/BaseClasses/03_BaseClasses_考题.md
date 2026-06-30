# BaseClasses 考题

## 🟢 概念题

**Q1: `DescriptionBaseSO` 的作用是什么？**

<details>
<summary>参考答案</summary>

为所有 SO 提供统一的描述字段，用于 Inspector 中的注释和文档。
</details>

---

**Q2: `SerializableScriptableObject` 的 Guid 是如何生成的？**

<details>
<summary>参考答案</summary>

在 `OnValidate()` 中通过 `AssetDatabase.AssetPathToGUID()` 将资产路径转换为唯一标识。
</details>

---

**Q3: 为什么 Guid 字段要 `HideInInspector`？**

<details>
<summary>参考答案</summary>

Guid 是内部实现细节，不需要在 Inspector 中显示或编辑。
</details>

---

**Q4: `OnValidate()` 什么时候被调用？**

<details>
<summary>参考答案</summary>

在 Editor 中资产被加载、修改或导入时调用。运行时不会调用。
</details>

---

**Q5: 为什么需要 Guid 而非直接序列化 SO 引用？**

<details>
<summary>参考答案</summary>

1. 跨场景持久化：SO 引用在场景中可能丢失
2. 存档友好：Guid 可以序列化为字符串
3. 地址兼容：支持 Addressables 系统的异步加载
</details>

---

## 🟡 机制题

**Q6: 如果两个 SO 资产有相同的 Guid，会发生什么？**

<details>
<summary>参考答案</summary>

存档系统会错误地加载错误的资产。Guid 必须全局唯一。
</details>

---

**Q7: 如果资产被移动或重命名，Guid 会改变吗？**

<details>
<summary>参考答案</summary>

不会。Guid 与资产路径绑定，Unity 在移动/重命名时会保持 Guid 不变。
</details>

---

**Q8: 运行时可以获取 Guid 吗？**

<details>
<summary>参考答案</summary>

可以。Guid 在 Editor 中生成并序列化，运行时可以读取。
</details>

---

**Q9: 为什么 `DescriptionBaseSO` 不继承 `SerializableScriptableObject`？**

<details>
<summary>参考答案</summary>

不是所有需要描述的 SO 都需要 Guid。分离关注点。
</details>

---

**Q10: `OnValidate()` 在构建后的游戏中会被调用吗？**

<details>
<summary>参考答案</summary>

不会。`OnValidate()` 只在 Editor 中调用。
</details>

---

**Q11: 如果 SO 资产被删除，Guid 会怎样？**

<details>
<summary>参考答案</summary>

Guid 随之消失。引用该 Guid 的存档会加载失败。
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`DescriptionBaseSO` 和 `SerializableScriptableObject` 的主要区别是什么？**

<details>
<summary>参考答案</summary>

- `DescriptionBaseSO`：提供描述字段
- `SerializableScriptableObject`：提供 Guid 序列化
</details>

---

**Q13: 【漏步后果】如果忘记在 `OnValidate()` 中生成 Guid，会有什么问题？**

<details>
<summary>参考答案</summary>

存档系统无法正确序列化/反序列化 SO 引用，导致加载失败或数据丢失。
</details>

---

**Q14: 【架构陷阱】如果两个不同的 SO 类型使用同一个 Guid，会有什么问题？**

<details>
<summary>参考答案</summary>

存档系统可能错误地加载错误类型的资产。
</details>

---

**Q15: 【同底座差异】Guid 和 InstanceID 有什么区别？**

<details>
<summary>参考答案</summary>

- Guid：资产级别，跨会话持久
- InstanceID：运行时对象级别，每次运行不同
</details>

---

**Q16: 【漏步后果】如果 SO 资产被复制但 Guid 未更新，会有什么问题？**

<details>
<summary>参考答案</summary>

两个资产有相同的 Guid，存档系统无法区分。
</details>

---

**Q17: 【架构陷阱】为什么不在运行时生成 Guid？**

<details>
<summary>参考答案</summary>

运行时生成的 Guid 每次运行不同，无法用于持久化引用。
</details>

---

**Q18: 【漏步后果】如果 `OnValidate()` 中使用了错误的 API，会有什么问题？**

<details>
<summary>参考答案</summary>

Guid 可能为空或错误，导致存档系统失败。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `WeaponSO`，继承 `SerializableScriptableObject`。**

<details>
<summary>参考答案</summary>

```csharp
[CreateAssetMenu(menuName = "Items/Weapon")]
public class WeaponSO : SerializableScriptableObject
{
    public string weaponName;
    public int damage;
    public Sprite icon;
}
```
</details>

---

**Q20: 实现一个扩展方法，通过 Guid 查找 SO 资产。**

<details>
<summary>参考答案</summary>

```csharp
public static class GuidExtensions
{
    public static T FindByGuid<T>(this string guid) where T : UnityEngine.Object
    {
        string path = UnityEditor.AssetDatabase.GUIDToAssetPath(guid);
        return UnityEditor.AssetDatabase.LoadAssetAtPath<T>(path);
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：存档加载时找不到资产。请列出至少 2 个可能的原因。**

<details>
<summary>参考答案</summary>

1. Guid 对应的资产已被删除或移动
2. Guid 在 `OnValidate()` 中未正确生成
3. 存档中的 Guid 被篡改
</details>
