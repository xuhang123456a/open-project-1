# StateMachine 考题

## 🟢 概念题

**Q1: `IStateComponent` 接口定义了哪些方法？为什么 `StateTransition` 要实现它？**

<details>
<summary>参考答案</summary>

`IStateComponent` 定义了 `OnStateEnter()` 和 `OnStateExit()`。`StateTransition` 实现它是因为在状态进入/退出时需要通知其包含的 `Condition` 执行相应的生命周期逻辑（如重置内部状态）。
</details>

---

**Q2: `StateAction` 的 `OnUpdate()` 为什么是抽象的，而 `OnStateEnter()` / `OnStateExit()` 是虚函数？**

<details>
<summary>参考答案</summary>

`OnUpdate()` 是状态在每一帧的核心行为，必须由子类实现（如移动、攻击）。`OnStateEnter()` / `OnStateExit()` 是可选的钩子，很多动作不需要进入/退出逻辑，所以提供默认空实现。
</details>

---

**Q3: `StateCondition` 是 `readonly struct` 而非 `class`，这有什么好处？**

<details>
<summary>参考答案</summary>

`StateCondition` 是轻量包装器（持有 Condition 引用 + bool 期望值），作为 struct 可以避免堆分配，减少 GC 压力。readonly 保证不可变性。
</details>

---

**Q4: `TransitionTableSO` 的 `GetInitialState()` 返回的是第一个 State，为什么？**

<details>
<summary>参考答案</summary>

因为 `_transitions` 数组按 FromState 分组，第一个分组的 Key 就是初始状态。这是约定：配置时第一个 FromState 必须是起始状态。
</details>

---

**Q5: `Condition` 的缓存机制是如何工作的？**

<details>
<summary>参考答案</summary>

- 首次调用 `GetStatement()` 时执行 `Statement()` 并缓存结果到 `_cachedStatement`
- 后续调用直接返回缓存值
- `ClearStatementCache()` 将 `_isCached` 设为 false，下一帧重新计算
</details>

---

## 🟡 机制题

**Q6: 如果两个 `State` 引用了同一个 `StateActionSO`，运行时会有几个 `StateAction` 实例？为什么？**

<details>
<summary>参考答案</summary>

只有1个。`TransitionTableSO.GetInitialState()` 使用 `Dictionary<ScriptableObject, object>` 作为 `createdInstances` 参数传递给所有 `GetState()` / `GetAction()` 调用，保证同一 SO 只创建一次。
</details>

---

**Q7: `resultGroups` 是如何处理 `A AND B OR C` 这样的条件的？**

<details>
<summary>参考答案</summary>

`ProcessConditionUsages()` 遍历 `ConditionUsage` 数组，将连续的 And 条件合并到同一组。例如 `[A,And,B,Or,C]` 会生成 `resultGroups = [2, 1]`，表示第一组2个条件（A AND B），第二组1个条件（C）。`ShouldTransition()` 内按组计算 AND，组间计算 OR。
</details>

---

**Q8: `StateMachine.Update()` 中先检查转换再执行 `OnUpdate()`，如果转换发生了，当前帧的 `OnUpdate()` 还会执行吗？**

<details>
<summary>参考答案</summary>

会。转换后 `_currentState` 已指向新状态，然后执行 `_currentState.OnUpdate()`，即新状态的 `OnUpdate()`。旧状态的 `OnUpdate()` 不会执行（因为已经 `OnStateExit()` 了）。
</details>

---

**Q9: `StateTransition.TryGetTransiton()` 返回 true 后，为什么还要调用 `ClearConditionsCache()`？**

<details>
<summary>参考答案</summary>

因为条件缓存是跨 Transition 共享的。如果第一个 Transition 检查后不清除，第二个 Transition 的 Condition 会返回缓存值（可能是上一帧的），导致逻辑错误。
</details>

---

**Q10: `GetOrAddComponent<T>()` 和 `TryGetComponent<T>()` 有什么区别？**

<details>
<summary>参考答案</summary>

- `TryGetComponent<T>()`：尝试获取组件，返回 bool 表示是否存在
- `GetOrAddComponent<T>()`：如果不存在则自动 `AddComponent<T>()`，保证返回非 null
</details>

---

**Q11: 为什么 `State` 的 `_transitions` 和 `_actions` 字段是 `internal` 而非 `public`？**

<details>
<summary>参考答案</summary>

封装性。这些字段只在 StateMachine 框架内部使用（`State` 类的方法访问），不需要暴露给外部。`internal` 限制为程序集内访问。
</details>

---

## � 架构陷阱题

**Q12: 【同底座差异】`StateTransition` 和 `StateAction` 都实现了 `IStateComponent`，它们的生命周期回调有什么区别？**

<details>
<summary>参考答案</summary>

- `StateTransition.OnStateEnter/Exit()`：通知其包含的 `Condition` 执行进入/退出逻辑（如重置计时器）
- `StateAction.OnStateEnter/Exit()`：执行动作自身的初始化/清理（如播放音效、设置标志位）
- 关键区别：Transition 的回调是"广播"给子组件，Action 的回调是"自身行为"
</details>

---

**Q13: 【漏步后果】如果忘记在 `ShouldTransition()` 之后调用 `ClearConditionsCache()`，会发生什么？**

<details>
<summary>参考答案</summary>

条件会一直返回缓存值。例如：第一帧条件为 true 触发转换，第二帧条件实际已变为 false，但由于缓存，仍然返回 true，导致状态机"卡"在错误的状态或反复触发转换。
</details>

---

**Q14: 【漏步后果】如果 `StateActionSO.GetAction()` 中忘记调用 `Awake(stateMachine)`，会有什么问题？**

<details>
<summary>参考答案</summary>

`StateAction` 中的组件引用不会被缓存。例如 `ChasingTargetAction` 在 `Awake()` 中获取 `NavMeshAgent`，如果忘记调用，`OnUpdate()` 中访问 `_agent` 会为 null，导致空引用异常。
</details>

---

**Q15: 【同底座差异】`Condition` 和 `StateCondition` 的关系是什么？为什么要分两层？**

<details>
<summary>参考答案</summary>

- `Condition`：抽象基类，定义具体的判断逻辑（`Statement()`）
- `StateCondition`：结构体，包装 `Condition` + 期望值（`_expectedResult`）
- 分离原因：同一个 `Condition` 实例可以被多个 Transition 使用，但期望值可能不同（一个要求 true，另一个要求 false）
</details>

---

**Q16: 【架构陷阱】如果 `TransitionTableSO` 中两个 `TransitionItem` 的 `FromState` 相同但 `ToState` 不同，运行时如何确定执行哪个转换？**

<details>
<summary>参考答案</summary>

按数组顺序遍历，第一个满足条件的 Transition 胜出（`break` 退出循环）。这意味着**顺序很重要**——应该把更具体的条件放在前面。
</details>

---

**Q17: 【漏步后果】如果 `StateSO.GetState()` 中忘记将新 State 加入 `createdInstances`，会怎样？**

<details>
<summary>参考答案</summary>

如果两个不同的 `TransitionItem` 引用同一个 `FromState`，会创建两个独立的 `State` 实例。这会导致：
1. 状态不一致（一个实例在 StateA，另一个在 StateB）
2. 内存浪费
3. 条件缓存失效（不同实例的 Condition 各自缓存）
</details>

---

**Q18: 【架构陷阱】`StateMachine` 的 `GetComponent<T>()` 在找不到组件时会抛出异常，而 Unity 原版返回 null。为什么这样设计？**

<details>
<summary>参考答案</summary>

这是"快速失败"策略。如果配置要求某个组件存在但实际缺失，立即抛出异常可以让问题在开发阶段暴露，而不是在运行时产生难以追踪的 bug。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `TimeElapsedCondition`，当状态停留超过指定秒数后返回 true。**

<details>
<summary>参考答案</summary>

```csharp
public class TimeElapsedCondition : Condition
{
    private float _duration;
    private float _enterTime;

    public TimeElapsedCondition(float duration)
    {
        _duration = duration;
    }

    public override void OnStateEnter() => _enterTime = Time.time;

    protected override bool Statement() => Time.time - _enterTime >= _duration;
}
```

关键点：必须在 `OnStateEnter()` 中记录进入时间，否则无法正确计时。
</details>

---

**Q20: 实现一个 `PatrolAction`，让实体在两个点之间循环移动。**

<details>
<summary>参考答案</summary>

```csharp
public class PatrolAction : StateAction
{
    private Transform _transform;
    private Vector3 _pointA, _pointB;
    private float _speed;
    private bool _goingToB;

    public PatrolAction(Vector3 a, Vector3 b, float speed)
    {
        _pointA = a;
        _pointB = b;
        _speed = speed;
    }

    public override void OnUpdate()
    {
        var target = _goingToB ? _pointB : _pointA;
        _transform.position = Vector3.MoveTowards(_transform.position, target, _speed * Time.deltaTime);
        if (Vector3.Distance(_transform.position, target) < 0.1f)
            _goingToB = !_goingToB;
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：状态机在 `StateA` 和 `StateB` 之间无限循环。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. **条件缓存未清除**：`ClearConditionsCache()` 未被调用，导致条件一直返回 true
2. **条件逻辑错误**：两个状态的条件互斥性不够（如 `A→B` 条件是 `true`，`B→A` 条件也是 `true`）
3. **OnStateEnter 修改了条件依赖的状态**：进入 StateA 时设置了某个标志，该标志恰好是 `A→B` 的触发条件
4. **多个 Transition 冲突**：同一状态有多个 Transition，第一个总是满足条件
</details>
