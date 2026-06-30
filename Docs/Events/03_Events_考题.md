# Events 模块考题

---

## 🟢 概念题（5）

1. 事件通道为什么做成 ScriptableObject 资产，而不是 C# `static event` 或全局单例？
2. `VoidEventChannelSO` 和 `IntEventChannelSO` 的本质区别是什么？
3. `VoidEventListener` 在整条链路里扮演什么角色？
4. `RuntimeAnchor` 解决了什么「跨场景引用」问题？
5. `LoadEventChannelSO.RaiseEvent` 在没有订阅者时与 `VoidEventChannelSO` 行为有何不同？为什么这样设计？

<details><summary>参考答案要点</summary>

1. 资产可被发布者/订阅者在 Inspector 同时引用，编译期零耦合且策划可配置；static event 无法在 Inspector 配置且全局唯一难复用；单例引入生命周期与初始化顺序问题。
2. 委托签名：`UnityAction`（无参）vs `UnityAction<int>`（带 int 载荷）。`RaiseEvent` 一个无参一个带参。
3. 适配器：把「跨场景广播的通道」转接到「本地 GameObject 上可在 Inspector 连线的 `UnityEvent` 响应」。
4. A 对象需引用运行时才生成、可能在别的场景的 B 对象（如玩家 Transform）。通过共享锚点资产间接持有，生成时 `Provide`，需要时读 `Value`/订阅 `OnAnchorProvided`。
5. Void 版静默；Load 版打印告警。因为「请求加载场景却无人接听」是严重错误（玩家会卡住），需立即暴露。
</details>

---

## 🟡 机制题（6）

1. `RaiseEvent` 里 `if (OnEventRaised != null)` 判空省略会怎样？
2. `OnEventRaised` 是 `public UnityAction` 字段而非 `public event UnityAction`。这两者在「外部可做什么」上差别是什么？
3. `AudioCueEventChannelSO.RaisePlayEvent` 为什么不能用 `UnityAction`，而要自定义 `delegate AudioCueKey AudioCuePlayAction(...)`？
4. Listener 为什么在 `OnEnable` 订阅而不是 `Awake`、在 `OnDisable` 退订而不是 `OnDestroy`？
5. 事件是同步还是异步派发？一个订阅者在回调里抛异常，对其他订阅者有何影响？
6. `RuntimeAnchorBase.OnDisable` 调 `Unset()`。这一步防止了什么跨 Play 会话的问题？

<details><summary>参考答案要点</summary>

1. 无订阅者时 `null.Invoke()` 抛 `NullReferenceException`。
2. 裸字段：外部可读、可整体赋值（`= null` 清空所有订阅）、可直接 `Invoke`。`event`：外部只能 `+=`/`-=`，不能赋值或 Invoke。原版选裸字段换灵活，牺牲封装。
3. `UnityAction`/`UnityAction<...>` 无返回值；play 需要回传 `AudioCueKey` 作控制句柄，故需带返回值的自定义委托——把「事件」升级为「请求-应答」。
4. OnEnable/OnDisable 与「对象激活/失活」对齐，失活对象不应再响应事件；Awake/OnDestroy 只在创建/销毁触发，无法处理「中途失活又启用」的对象，会导致失活期间仍被回调或重复订阅。
5. 同步、按订阅顺序在调用线程依次执行。某订阅者抛异常会中断 `Invoke` 委托链，**后续订阅者收不到**该次事件（多播委托异常传播特性）。
6. SO 是资产，跨 Play 会话字段残留。不 Unset 的话上次 Play 残留的 `_value`（已销毁对象）和 `isSet=true` 会被下次 Play 误读为「已就绪」，导致引用悬空对象。
</details>

---

## 🔴 架构陷阱题（7）

1. 【同底座差异】事件通道与 RuntimeAnchor 都是「共享 SO + 委托通知」。二者语义有何根本不同？什么场景该用通道、什么场景该用锚点？
2. 【漏步后果】某 MonoBehaviour 在 `OnEnable` 订阅了通道但忘了在 `OnDisable` 退订，对象被销毁。之后另一处 `RaiseEvent`，会发生什么？
3. 【裸字段陷阱】两个系统 A、B 都监听同一个通道。某第三方代码误写 `channel.OnEventRaised = MyHandler`（用 `=` 而非 `+=`）。后果是什么？这种 bug 用 `event` 关键字能否在编译期挡住？
4. 【请求-应答的单应答约束】`AudioCueEventChannelSO` 的 play 通道若同时被两个 AudioManager 订阅（`+=` 两次），`RaisePlayEvent` 的返回值是哪个？多播委托带返回值时返回什么？这会引出什么 bug？
5. 【顺序依赖】事件同步派发，订阅顺序 = 注册顺序。若系统 B 的正确响应依赖系统 A 已先响应完，但 A 比 B 晚 `OnEnable`，会怎样？这暴露了纯事件解耦的什么局限？
6. 【锚点时序】相机在 `Start` 直接读 `playerAnchor.Value`，但玩家由 SpawnSystem 在某事件后才 `Provide`。若相机的 Start 早于 Provide，会怎样？正确写法是什么？
7. 【调试黑洞】事件解耦后，「谁触发了这次场景加载」在代码里无法静态追踪。项目用什么约定缓解？这种解耦的代价是什么？

<details><summary>参考答案要点</summary>

1. 通道是**瞬时广播**（fire-and-forget，无状态留存，错过即错过）；锚点是**持久引用槽**（有状态 `Value`/`isSet`，晚到的消费者仍能读到当前值）。需要「通知一次性事件」用通道；需要「随时取最新的某个对象引用」用锚点。
2. 通道委托链仍持有已销毁 MonoBehaviour 的引用，`Invoke` 时触发 `MissingReferenceException`（Unity 重写了 `==`，对象虽逻辑销毁但委托仍引用），且阻断后续订阅者。
3. `=` 覆盖了 A、B 的订阅，只剩 `MyHandler`，A、B 静默失效。用 `event` 关键字则 `channel.OnRaised = ...` 编译报错（event 外部不可赋值），能挡住。
4. 多播委托带返回值时**只返回最后一个被调用委托的返回值**，前面的返回值被丢弃。两个 AudioManager 都会执行 play（声音播两遍），但只有最后注册者的 `AudioCueKey` 被返回——另一个 Manager 创建的 emitter 句柄丢失，无法 Stop，泄漏。故请求通道隐含「单一应答者」约束。
5. B 可能在 A 未就绪时响应，读到 A 的旧状态/空状态，产生错误。暴露局限：纯事件无法表达「响应顺序/依赖」，需要额外的初始化时序保证（如 PersistentManagers 场景先加载）。
6. 相机读到 `Value==null`/`isSet==false`，相机无目标（NullRef 或不跟随）。正确：订阅 `OnAnchorProvided` 在回调里绑定，或每帧检查 `isSet` 再用——利用锚点「持久 + 通知」双能力解耦时序。
7. 项目用字段上的 `[Header("Listening to")]` / `[Header("Broadcasting on")]` 注释 + 通道资产命名约定自文档化。代价：失去编译期可追踪性，跨场景数据流要靠文档/命名/运行时日志理解，重构风险高。
</details>

---

## ✍️ 实操题（3）

1. 现有通道用裸字段。把 `VoidEventChannelSO` 改造为「只允许 += / -=，禁止外部赋值与 Invoke」，写出关键改动，并说明会破坏哪些现有用法。
2. 设计一个「带历史回放」的通道：新订阅者订阅时立即收到最近一次事件的载荷（类似 BehaviorSubject）。基于 `EventChannel<T>` 给出最小实现。
3. `RuntimeAnchor` 当前只在 `Provide` 时通知。若消费者在 `Provide` 之后才订阅 `OnAnchorProvided`，它收不到通知却 `isSet==true`。设计一个 API 让「迟到订阅者」也能被立即触发一次。

<details><summary>参考答案要点</summary>

1. 把 `public UnityAction OnEventRaised;` 改为 `private UnityAction _onRaised; public event UnityAction OnEventRaised { add{...} remove{...} }`，`RaiseEvent` 内部调 `_onRaised`。破坏：任何依赖「直接赋值/外部 Invoke/反射清空」的代码，以及某些把通道当裸委托用的测试。
2. `EventChannel<T>` 加 `bool _hasValue; T _last;`，`Raise` 时缓存 `_last=payload;_hasValue=true`；提供 `Subscribe(Action<T> h)` 在内部 `OnRaised+=h` 后若 `_hasValue` 立即 `h(_last)`。
3. 加 `void Register(Action onProvided){ OnAnchorProvided += onProvided; if(IsSet) onProvided(); }`——订阅同时若已置位则补触发一次。
</details>
