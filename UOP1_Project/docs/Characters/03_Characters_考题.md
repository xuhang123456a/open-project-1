# Characters 考题

## 🟢 概念题

**Q1: `Damageable` 的 `GetHit` 标志有什么作用？**

<details>
<summary>参考答案</summary>

防止同一攻击多次造成伤害。`Attack.OnTriggerEnter()` 检查 `!damageable.GetHit`。
</details>

---

**Q2: `HealthSO` 为什么是 ScriptableObject 而非普通类？**

<details>
<summary>参考答案</summary>

1. 可以在 Inspector 中配置初始值
2. 支持数据共享（如玩家预设）
3. 支持运行时创建（如敌人）
</details>

---

**Q3: `Critter` 的 `OnAlertTriggerChange` 和 `OnAttackTriggerChange` 有什么区别？**

<details>
<summary>参考答案</summary>

- `OnAlertTriggerChange`：外圈警戒区，设置/清除目标
- `OnAttackTriggerChange`：内圈攻击区，只更新标志
</details>

---

**Q4: `ZoneTriggerController` 的 `LayerMask` 有什么作用？**

<details>
<summary>参考答案</summary>

过滤触发对象，只响应指定层的物体（如玩家层）。
</details>

---

**Q5: `Attack` 为什么在 `Awake()` 中禁用？**

<details>
<summary>参考答案</summary>

攻击碰撞体只在攻击时启用，避免持续触发伤害。
</details>

---

## � 机制题

**Q6: 如果 `Damageable.Awake()` 中不创建 `HealthSO`，敌人会怎样？**

<details>
<summary>参考答案</summary>

`_currentHealthSO` 为 null，`ReceiveAnAttack()` 中空引用异常。
</details>

---

**Q7: `Critter` 如何处理目标死亡？**

<details>
<summary>参考答案</summary>

订阅 `currentTarget.OnDie`，在 `OnTargetDead()` 中清除引用和标志。
</details>

---

**Q8: `DropGroup` 的掉落概率是如何计算的？**

<details>
<summary>参考答案</summary>

1. 先检查 `groupDropRate`（组掉落率）
2. 如果命中，遍历组内物品，累加 `dropRate` 选择物品
</details>

---

**Q9: `Attack` 如何避免友军伤害？**

<details>
<summary>参考答案</summary>

检查 `other.CompareTag(gameObject.tag)`，标签相同则忽略。
</details>

---

**Q10: `Protagonist` 的 `movementInput` 和 `movementVector` 有什么区别？**

<details>
<summary>参考答案</summary>

- `movementInput`：初始输入（相机方向 + 加速度）
- `movementVector`：最终移动向量（被 StateMachine Actions 修改）
</details>

---

**Q11: `MovingCritterAttackController` 是如何实现冲撞的？**

<details>
<summary>参考答案</summary>

1. `SetAttackTarget()` 计算冲撞方向
2. `AttackPropelTrigger()` 启动计时器
3. `Update()` 中按固定速度移动
</details>

---

## 🔴

**Q12: 【同底座差异】`Damageable` 和 `HealthSO` 为什么要分离？**

<details>
<summary>参考答案</summary>

- `HealthSO`：纯数据，可共享
- `Damageable`：逻辑组件，处理受伤、死亡、事件
</details>

---

**Q13: 【漏步后果】如果 `Critter` 在 `OnAlertTriggerChange` 中忘记取消订阅 `OnDie`，会有什么问题？**

<details>
<summary>参考答案</summary>

内存泄漏。已死亡的目标仍被引用。
</details>

---

**Q14: 【架构陷阱】如果两个 `Attack` 碰撞体同时触发同一个 `Damageable`，会发生什么？**

<details>
<summary>参考答案</summary>

第一个触发设置 `GetHit = true`，第二个被忽略（`!damageable.GetHit` 检查）。
</details>

---

**Q15: 【同底座差异】`ZoneTriggerController` 和 `OnTriggerEnter` 直接处理有什么区别？**

<details>
<summary>参考答案</summary>

- `ZoneTriggerController`：通用化，支持 Inspector 配置 LayerMask 和事件
- 直接处理：硬编码，不灵活
</details>

---

**Q16: 【漏步后果】如果 `DropGroup` 的 `groupDropRate` 检查后忘记 `break`，会有什么问题？**

<details>
<summary>参考答案</summary>

可能连续掉落多个组的物品。
</details>

---

**Q17: 【架构陷阱】`Protagonist` 的输入消费与 StateMachine 的状态如何配合？**

<details>
<summary>参考答案</summary>

- `Protagonist`：消费输入，设置标志（`jumpInput`、`attackInput`）
- StateMachine Conditions：读取标志，决定状态转换
- StateMachine Actions：读取 `movementInput`，修改 `movementVector`
</details>

---

**Q18: 【漏步后果】如果 `Attacker` 在攻击动画结束前禁用武器，会有什么问题？**

<details>
<summary>参考答案</summary>

攻击判定提前结束，可能导致攻击无效。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `HealthBarUI`，监听 `Damageable` 的生命值变化。**

<details>
<summary>参考答案</summary>

```csharp
public class HealthBarUI : MonoBehaviour
{
    [SerializeField] private Damageable _damageable;
    [SerializeField] private Image _healthBarFill;
    
    private void OnEnable() => _damageable.OnHealthChanged += UpdateUI;
    private void OnDisable() => _damageable.OnHealthChanged -= UpdateUI;
    
    private void UpdateUI(float current, float max)
    {
        _healthBarFill.fillAmount = current / max;
    }
}
```
</details>

---

**Q20: 实现一个 `ExplosionAttack`，对范围内所有 `Damageable` 造成伤害。**

<details>
<summary>参考答案</summary>

```csharp
public class ExplosionAttack : MonoBehaviour
{
    float _radius = 5f;
    [SerializeField] private int _damage = 50;
    
    public void Explode()
    {
        var hits = Physics.OverlapSphere(transform.position, _radius);
        foreach (var hit in hits)
        {
            if (hit.TryGetComponent(out Damageable d) && !d.IsDead)
            {
                d.ReceiveAnAttack(_damage);
            }
        }
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：敌人死亡后仍然攻击玩家。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. 死亡状态未正确设置 `IsDead = true`
2. StateMachine 未转换到死亡状态
3. `Attack` 碰撞体未禁用
</details>
