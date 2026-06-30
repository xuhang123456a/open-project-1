# StateMachine 模块考题

---

## 🟢 概念题（5）

1. 为什么要把「状态/转换/条件/行为」的定义放在 SO 层，而运行时另建一套 `State`/`StateTransition`/`Condition`/`StateAction`？
2. `TransitionTableSO.GetInitialState` 在什么时机被调用？它产出什么？
3. 用户要扩展一个新行为，需要写哪两个类？泛型版 `StateActionSO<T>` 帮你省了什么？
4. `StateCondition` 为什么是 `readonly struct` 而不是 class？
5. `StateMachine.Update` 里「先判转换、后执行 OnUpdate」，转换发生那一帧执行的是哪个状态的 OnUpdate？

<details><summary>参考答案要点</summary>

1. 数据层可被策划在 Inspector 配置、可被多个状态机实例共享（享元）；运行时层持有缓存/目标引用等可变外蕴状态，轻量且每实例独立。分离让「配置」与「执行」解耦。
2. `StateMachine.Awake` 调用一次，构建整张运行时图并返回初始 `State`。
3. 派生 `StateAction`（写逻辑）+ 建 `StateActionSO<MyAction>` 资产。泛型版自动实现 `CreateAction() => new T()`，省去手写工厂方法。
4. 它是「条件+期望值」的轻量不可变快照，按值传递避免堆分配且语义清晰（不可变）。
5. 新状态。因为 Transition 内已完成「旧 OnExit → 切换 → 新 OnEnter」，紧接的 `_currentState.OnUpdate()` 跑的是新状态。
</details>

---

## 🟡 机制题（6）

1. `createdInstances` 字典的 key 和 value 各是什么？它解决了什么重复问题？
2. `Condition.GetStatement` 的缓存在什么时候被设置、什么时候被清除？为什么不能每条 transition 求值后立刻清？
3. `StateSO.GetState` 里 `createdInstances.Add(this, state)` 出现在「构建 actions 之前」。把它挪到「构建 actions 之后」会有什么风险？
4. `_resultGroups = [2, 1]` 表示怎样的条件逻辑？对应几个 OR 组、各几个 AND？
5. `StateMachine.TryGetComponent<T>` 相比 Unity 原生 `GetComponent<T>` 多做了什么？为什么 Action 应在 `Awake` 而非 `OnUpdate` 里取组件？
6. `#if UNITY_EDITOR` 下的 `OnAfterAssemblyReload` 重新调 `GetInitialState`。为什么编辑器里热重载后必须重建运行时图？

<details><summary>参考答案要点</summary>

1. key = 定义 SO（`ScriptableObject`），value = 对应运行时实例（`object`：State/Action/Condition）。解决「同一 SO 被图中多处引用时重复实例化」，保证全图唯一运行时实例（享元）。
2. `GetStatement` 首次求值时设缓存；`State.TryGetTransition` 遍历完**全部** transition 后统一 `ClearConditionsCache`。不能每条立刻清，否则被多条 transition 共享的同一条件会重复求值，破坏「单帧只算一次」优化。
3. 状态图有环（A 的转换目标含 A 自身或经 B 回到 A）时，先建 actions 再入缓存会导致构建该状态时再次进入 `GetState` 找不到缓存 → 无限递归 / 重复 new。先入缓存可让递归命中。
4. 两个 OR 组：第一组 2 个 AND 条件（c0 AND c1），第二组 1 个条件（c2）。整体 =（c0 AND c1）OR c2。
5. 多了 `_cachedComponents` 字典缓存 `Type→Component`，避免重复 `GetComponent`（Unity 的 GetComponent 有遍历开销）。Action 在 Awake 取一次缓存，OnUpdate 每帧直接用，避免每帧 GetComponent 的性能损耗。
6. 热重载会丢失/重置托管对象状态（运行时图是普通 C# 对象，不被序列化），`_currentState` 等会失效，必须重建以恢复一致的运行时图。
</details>

---

## 🔴 架构陷阱题（8）

1. 【享元后果】状态 A 和状态 B 都引用同一个 `MoveActionSO`。运行时它们用的是同一个 `MoveAction` 实例还是各一个？若是同一个，A、B 都进入时 `OnStateEnter` 被调用几次？这会引出什么共享状态 bug？
2. 【漏步后果】自定义 Action 在 `OnStateEnter` 里订阅了某事件，但忘了在 `OnStateExit` 退订。状态频繁进出会怎样？
3. 【缓存时序】假设错误地实现成「每条 transition 求值后立即 `ClearConditionsCache`」。一个条件 `IsGrounded` 被 3 条 transition 共享，本帧 `IsGrounded` 是个昂贵计算。后果是什么？正确性会受影响吗？
4. 【与/或边界】`ProcessConditionUsages` 里 `while (i < count-1 && Operator == And)`。如果最后一个条件的 `Operator` 被误设为 `And`（但它后面没有条件了），会发生什么？分组结果是否正确？
5. 【同底座差异】对比 Condition 的缓存（`GetStatement`/`ClearStatementCache`）与 Pool 的复用栈：两者都是「避免重复工作」，但缓存生命周期粒度有何根本不同？
6. 【构建 vs 运行】整张运行时图在 `Awake` 一次性构建。若某状态在游戏中从未被进入，它的 Actions 是否也被实例化和 `Awake`？有什么代价？能否改成懒构建？会破坏什么？
7. 【转换求值短路】`State.TryGetTransition` 遍历 transition，命中第一个满足的就 `break`。这意味着 transition 的**声明顺序**即优先级。若两条 transition 同帧都满足条件，哪条生效？策划把高优先级转换放在后面会怎样？
8. 【内蕴/外蕴】`StateAction._originSO` 指回定义 SO。若 Action 把每实例独有的可变状态（如一个计时器）误存到 `OriginSO`（SO 字段）上而非 Action 自身字段，多个状态机共享该 SO 时会怎样？

<details><summary>参考答案要点</summary>

1. 同一个实例（享元去重）。A、B 都进入时各触发一次 `OnStateEnter`（共 2 次，分属不同时刻）。bug：若该 Action 内部有「进入态」假设（如 enter 时 +1、exit 时 -1 配对），A 进入未退出时 B 又进入，计数/状态错乱——共享实例不持有「我属于哪个状态」的概念。
2. 通道委托链不断累积重复订阅（每次 enter 都 +=），且失活后仍持有引用。表现为：同一事件被响应多次、内存泄漏、悬空回调。必须 enter/exit 成对订阅退订。
3. 该帧 `IsGrounded` 被求值 3 次（每条 transition 求值后缓存被清，下条重新算）——性能退化为 3 倍昂贵计算。正确性不受影响（同帧物理状态一致），但违背了缓存的设计意图。
4. `while` 条件 `i < count-1` 保证不越界：最后一个条件即使 Operator=And 也不会再吞下一个（没有下一个）。它自成一组（groups 末尾 +1）。结果仍正确——And 作为「与下一条的连接符」，无下一条则无效。
5. 条件缓存是**单帧粒度**（每帧清，跨帧重算，反映实时变化）；Pool 复用栈是**对象生命周期粒度**（对象一直存活直到池清理）。前者缓存「计算结果」求实时性，后者复用「对象实例」求省分配。
6. 是，全部 Action 都被 new + Awake，无论是否会进入该状态。代价：启动时一次性分配 + Awake 开销（如组件缓存）。可改懒构建（首次进入才建），但会破坏：①享元缓存需贯穿整个运行期而非构建期；②首次进入有卡顿；③`GetInitialState` 当前依赖一次性遍历建图，懒构建需重构转换目标的延迟解析。
7. 第一条（声明靠前者）生效。声明顺序 = 优先级。高优先级放后面 → 永远被前面的低优先级抢先 `break`，高优先级转换失效。
8. SO 字段是所有共享该 SO 的状态机/状态实例**全局共享**的。把每实例计时器存 SO 上 → 多个状态机互相覆盖同一计时器，行为串台。可变外蕴状态必须存运行时 Action 实例字段，只有不变配置（内蕴）才放 SO。
</details>

---

## ✍️ 实操题（3）

1. 给 `StateTransition` 增加「全局转换」能力：无论当前在哪个状态，只要某条件满足就跳到目标状态（如「死亡」）。说明你会改 `TransitionTableSO` 构建逻辑的哪一步。
2. 当前条件缓存是 bool 单帧缓存。设计一个支持「条件依赖外部黑板变量」的方案：当黑板变量未变时跨帧复用缓存。指出需要新增的失效信号。
3. 写一个自定义 `Condition` 子类 `TimerCondition`：进入状态 N 秒后返回 true。说明计时器状态该存在哪里、为什么不能存 `OriginSO`，以及如何在 `OnStateEnter` 重置。

<details><summary>参考答案要点</summary>

1. 在 `GetInitialState` 构建每个 state 的 transitions 后，把一组「全局转换」追加到**每一个** state 的 `_transitions` 数组（或在 StateMachine.Update 里独立先判全局转换）。关键是这些全局转换的目标 State 仍要走 `createdInstances` 去重复用。
2. 给 `Condition` 加 `int _cachedFrameOrVersion`，黑板变量改变时递增一个版本号；`GetStatement` 比较当前版本与缓存版本，相等则复用，不等才重算。失效信号 = 黑板写入时的版本自增（替代「每帧无条件清缓存」）。
3. 计时器（已进入时长/进入时间戳）存 `TimerCondition` 实例字段（运行时外蕴状态），不能存 `OriginSO`——SO 被多个状态机共享会串台。实现 `OnStateEnter` 重置 `_elapsed=0`/记录 enterTime，`Statement()` 返回 `elapsed >= threshold`，threshold 可读自 `OriginSO`（不变配置）。
</details>
