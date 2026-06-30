# Pool 模块考题

> 陷阱题优先考察「同底座两模块的差异」与「漏掉某步的后果」。

---

## 🟢 概念题（5）

1. `PoolSO<T>` 为什么继承自 `ScriptableObject` 而不是普通 class 或 MonoBehaviour？
2. `IPool<T>` 定义了哪三个方法？各自职责是什么？
3. `Available` 用 `Stack<T>` 而非 `Queue<T>`，对复用顺序有何影响？
4. `Factory` 在池中扮演什么角色？为什么池自己不直接 `new`？
5. `ComponentPoolSO<T>` 的泛型约束 `where T : Component` 解决了什么问题？

<details><summary>参考答案要点</summary>

1. 让池成为一份可被多处 `[SerializeField]` 引用的**磁盘资产**，从而实现「无 Manager 的全局共享」；SO 也能在 Inspector 配置工厂引用与初始大小。
2. `Prewarm(int)` 预热批量创建；`Request()` 借出；`Return(T)` 归还。
3. LIFO：最近归还的对象最先被复用，缓存局部性更好；与正确性无关（池不关心顺序）。
4. 工厂封装「如何创建」；池只管「借还」。职责分离让同一池骨架可换不同造物策略（`new`+`AddComponent` vs `Instantiate(prefab)`）。
5. 只有 Component 才有 `gameObject`/`transform`，才能做 `SetActive`、`SetParent` 这些「激活态 + 层级」管理。
</details>

---

## 🟡 机制题（6）

1. `Prewarm` 被调用两次会发生什么？源码靠什么字段保证？
2. `Request()` 在池空时的行为，和池非空时有什么本质区别（内存角度）？
3. `Request(int num)` 每次调用都做了一次什么分配？批量借 1000 次会有什么隐性开销？
4. `ComponentPoolSO.Create()` 相比 `PoolSO.Create()` 多做了哪两步？为什么必须在 Create 时就做？
5. `ComponentPoolSO.OnDisable()` 做了什么？为什么用 `#if UNITY_EDITOR` 分支区分 `DestroyImmediate` 和 `Destroy`？
6. `_poolRoot` 是懒加载的（getter 里 `if (_poolRoot == null) new GameObject`）。第一次访问 `PoolRoot` 是在哪个时机？

<details><summary>参考答案要点</summary>

1. 第二次只打印警告并 `return`，不重复填充。靠 `HasBeenPrewarmed` 守卫。
2. 命中（Pop）是**零分配**复用；未命中走 `Create()` 是**新分配**。池的价值正在于把分配压到预热阶段，运行时尽量零分配。
3. 每次 `new List<T>(num)`——一个 List 容器的分配。高频调用会产生 GC 压力（返回的是 `IEnumerable`，但内部总是新建 List）。
4. 多做 `SetParent(PoolRoot)` + `SetActive(false)`。必须在 Create 时就失活，否则新建对象会以激活态参与渲染/Update，破坏「池中对象皆挂起」不变量。
5. 先 `base.OnDisable()`（清栈+复位标记），再销毁 `_poolRoot` 整个层级（连带所有子物体）。编辑器模式用 `DestroyImmediate`（非播放态 `Destroy` 不立即生效），运行时用 `Destroy`。
6. 任意首次访问——通常是首次 `Create()`/`Return()`/`SetParent()`。懒加载避免在没用到池时凭空创建一个空 GameObject。
</details>

---

## 🔴 架构陷阱题（7）

1. 【同底座差异】`PoolSO` 与 `ComponentPoolSO` 在「对象空闲态」的表示上有何根本不同？把一个普通 `PoolSO<SomePlainClass>` 当 ComponentPool 用会缺什么？
2. 【漏步后果】调用方 `Request()` 后忘记 `Return()`，池会怎样？池能自动回收吗？长期运行会出现什么现象？
3. 【double free】对同一个 SoundEmitter 连续 `Return()` 两次，原版源码会发生什么？后续 `Request()` 会暴露什么 bug？
4. 【共享语义】两个不同场景的对象都 `[SerializeField]` 引用同一个 `ParticlePoolSO.asset`。它们用的是同一个池还是各自一个？为什么？这与「new 一个 Pool」有何本质区别？
5. 【生命周期传染】`SetParent` 注释提到：把池根挂到 `DontDestroyOnLoad` 对象下会怎样？这是 bug 还是特性？如何撤销？
6. 【预热陷阱】`AudioManager.Awake` 里先 `Prewarm` 再 `SetParent`。如果交换顺序（先 SetParent 后 Prewarm），结果有何不同？如果两个 AudioManager 引用同一个池 SO 都调 Prewarm 呢？
7. 【SO 状态残留】SO 是磁盘资产，其字段在编辑器内跨 Play 会话可能残留。`OnDisable` 复位了 `HasBeenPrewarmed` 和清空 `Available`——若没有这步，编辑器里反复进入/退出 Play 会出现什么？

<details><summary>参考答案要点</summary>

1. `PoolSO` 仅靠「在不在 `Available` 栈里」表示空闲；`ComponentPoolSO` 额外用「`SetActive(false)` + 挂在 `_poolRoot` 下」作为可见的空闲态。普通类池没有激活/层级概念，借出的对象不会自动停止行为，需调用方自管「启用/挂起」。
2. 对象脱离池管理（Leaked），池不追踪借出对象，无法自动回收。长期运行表现为：每次 Request 都未命中→不断 `Create` 新对象→内存/实例数膨胀，池形同虚设。
3. 原版 `Return` 直接 `Push`，该对象在栈中出现两次。后续两次 `Request` 会把同一个 SoundEmitter 借给两个调用方，二者同时操作同一 AudioSource，声音/状态互相踩踏（典型 double free / aliasing）。
4. 同一个池。SO 资产在内存中是单实例，所有引用指向它，`Available` 栈共享。与 `new Pool()` 的区别：`new` 每次得到独立实例（独立栈），而 SO 引用是「指向同一份资产」。
5. 整个池（含其所有池化对象）会跟着 `DontDestroyOnLoad` 跨场景存活。是特性（可做全局持久池），也可能是隐患（忘记清理→泄漏）。撤销：`SetParent(null)` 或手动 `Destroy`。
6. 顺序在原版里基本等价（Prewarm 内部首次 Create 会懒创建 _poolRoot，SetParent 再重挂）。真正的坑是**两个 Manager 共享同一池 SO 都调 Prewarm**：第二次 Prewarm 被 `HasBeenPrewarmed` 拦截只警告——即「共享池只被预热一次」，第二个 Manager 以为自己预热了其实没有。
7. 没有 `OnDisable` 复位的话，退出 Play 时栈里残留上次的（已被销毁的）对象引用、`HasBeenPrewarmed` 仍为 true，下次 Play 会 `Request` 到已销毁对象或拒绝预热——出现 `MissingReference` 或池规模错乱。
</details>

---

## ✍️ 实操题（3）

1. 给原版 `PoolSO<T>` 增加一个 `int CountAvailable` 只读属性与 `int CountInUse` 统计（提示：需要引入借出计数），不破坏现有 API。写出关键改动。
2. 现有 `Return` 无幂等保护。设计一个最小改动，使重复归还同一对象被安全忽略，并说明它的内存代价。
3. `ParticlePoolSO` 的 `Factory` getter/setter 用了 `value as ParticleFactorySO`。请说明：若 Inspector 里拖入的工厂资产类型不匹配，setter 会发生什么？如何在运行时尽早暴露这个错误？

<details><summary>参考答案要点</summary>

1. 加 `private int _inUse;`，`Request` 时 `_inUse++`，`Return` 时 `_inUse--`；`CountAvailable => Available.Count`，`CountInUse => _inUse`。注意 `Request(int)` 走单个 `Request` 故计数自动正确。
2. 引入 `HashSet<T> _inPool`，`Return` 先判 `Contains` 再 `Push`/`Add`，`Request` 时 `Remove`。内存代价：一个 HashSet + 每对象一条哈希记录（O(n) 额外空间）。
3. `as` 失败返回 `null`，`_factory` 被静默置空；之后首次 `Create()` 调 `_factory.Create()` 抛 `NullReferenceException`。尽早暴露：在 `Prewarm`/首次 `Request` 前断言 `Factory != null`，或在 setter 里用强制转型 `(ParticleFactorySO)value` 让类型不匹配立即抛异常。
</details>
