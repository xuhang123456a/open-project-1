# SceneManagement 模块考题

---

## 🟢 概念题（5）

1. 场景为什么经 `GameSceneSO.sceneReference`（Addressables `AssetReference`）加载，而不是 `SceneManager.LoadScene("name")`？
2. `LoadEventChannelSO` 在加载链路里起什么作用？谁发、谁收？
3. `_onSceneReady`（VoidEventChannelSO）何时被广播？哪些下游会响应？
4. `GameSceneSO` 携带哪三类关键信息？`LocationSO` 在其上扩展了什么？
5. `InitializationLoader` 与 `SceneLoader` 的职责分工是什么？

<details><summary>参考答案要点</summary>

1. AssetReference 让场景成为可按需打包/下载的 Addressable 资源，并提供 `AsyncOperationHandle` 用于后续精确卸载；字符串名加载无法管理资源句柄与打包。
2. 解耦中枢：`LocationExit`/`StartGame`/`InitializationLoader` 发（RaiseEvent），`SceneLoader` 是唯一订阅者收。
3. `OnNewSceneLoaded` 完成后经 `StartGameplay()` 广播。下游：SpawnSystem（生成玩家）、`LocationEntrance`（相机转场）。
4. `sceneType`（场景类型枚举）、`sceneReference`（加载句柄来源）、`musicTrack`（关联音乐）。`LocationSO` 加 `LocalizedString locationName`。
5. `InitializationLoader` 负责冷启动引导：加载常驻管理器场景 + 触发主菜单加载，然后卸载自身（index 0）。`SceneLoader` 负责后续所有 Location/Menu 的加载卸载编排。
</details>

---

## 🟡 机制题（6）

1. `_isLoading` 在何处置 true、何处置 false？它防的是什么具体场景（源码注释里说了）？
2. `UnloadPreviousScene` 为什么先 `FadeOut` 并 `WaitForSeconds` 再卸场景，而不是立即卸？
3. 从主菜单进入 Location 时，`LoadLocation` 为什么要先检查并加载 `_gameplayScene`？返回主菜单时又对它做了什么？
4. `OnNewSceneLoaded` 里 `SceneManager.SetActiveScene(s)` 和 `LightProbes.TetrahedralizeAsync()` 各解决什么问题？
5. 卸载旧场景时为何要判断 `sceneReference.OperationHandle.IsValid()`？不成立时（`#if UNITY_EDITOR`）走的是哪条路径、为什么？
6. `StartGame.ContinuePreviousGame` 如何从存档恢复到正确的 Location？涉及哪种「键」？

<details><summary>参考答案要点</summary>

1. 在 `LoadLocation`/`LoadMenu` 入口置 true，在 `OnNewSceneLoaded` 末尾置 false。防：玩家一帧内同时落入两个 Exit 触发器导致并发双重加载。
2. 为视觉过渡——先黑屏（FadeOut）再卸场景，玩家看不到场景突然消失/穿帮；fade 时长内还顺便禁用输入。
3. Gameplay 是常驻管理器场景（含 spawn 等系统），玩法期间必须存在；主菜单时它不该存在。进 Location 按需 Additive 加载它；回主菜单 `Addressables.UnloadSceneAsync` 卸掉它。
4. `SetActiveScene` 让新场景成为 lighting/实例化的默认目标场景；`TetrahedralizeAsync` 异步重建光照探针网格，使新场景光照正确（合并 additive 场景的探针）。
5. 判断该场景是否经 Addressables 加载过（持有有效 handle）。冷启动时场景已在编辑器中打开、未经 Addressables 加载，handle 无效，故用原生 `SceneManager.UnloadSceneAsync(name)` 卸载。
6. 用存档里的 `_locationId`（LocationSO 的 GUID），经 `Addressables.LoadAssetAsync<LocationSO>(guid)` 加载回 SO，再 RaiseEvent 加载该场景。涉及 GUID 资源键。
</details>

---

## 🔴 架构陷阱题（7）

1. 【漏步后果】若 `_isLoading` 的复位被错误地放在 `LoadNewScene`（加载启动时）而非 `OnNewSceneLoaded`（加载完成时），会出现什么并发问题？
2. 【永久卡死】加载流程中某异步步骤抛异常导致 `OnNewSceneLoaded` 没被调到，`_isLoading` 停在 true。之后所有加载请求会怎样？如何用 `try/finally` 规避？
3. 【句柄配对】加载用 handle A，但卸载时误用一个新建的 handle B（或重复卸载同一 handle）。Addressables 会怎样？为什么源码要把 handle 缓存成字段？
4. 【同底座差异】对比 `_loadLocation` 与 `_loadMenu` 两个 LoadEventChannel：它们对「常驻 Gameplay 场景」的处理正好相反。分别是什么？搞反会怎样？
5. 【唯一订阅者约束】`LoadEventChannelSO` 隐含「只有 SceneLoader 订阅」。若某调试脚本也订阅了同一个加载通道，会发生什么？为什么这个约束不在代码里强制？
6. 【冷启动陷阱】开发者在编辑器里直接 Play 一个 Location 场景（跳过 Initialization）。若没有 `LocationColdStartup` 这条特殊路径，会缺什么、表现为什么 bug？
7. 【时序竞态】`LocationEntrance.Awake` 订阅 `_onSceneReady`，而 `SceneLoader` 在加载完成时广播它。若 Entrance 所在场景比 SceneLoader 广播更晚激活（Awake 晚于广播），相机转场会怎样？锚点/事件哪种更适合此类「可能错过」的通知？

<details><summary>参考答案要点</summary>

1. 加载尚未完成时闸门已开，途中到来的第二个请求会插入并与第一个并发执行——两条加载流水线同时卸载/加载场景，`_currentlyLoadedScene` 等状态互相覆盖，场景错乱或重复加载。
2. 闸门永久为 true，后续每个请求开头 `if(_isLoading) return;` 全被吞掉，游戏再也无法切场景（卡死）。规避：把加载主体放 `try`，`_isLoading=false` 放 `finally`，确保任何出口都复位。
3. 卸载用错 handle → 目标资源未被释放（泄漏）或抛「handle 无效」异常；重复卸载同一 handle → double-free 异常。缓存成字段是为了「加载时拿到的 handle」能在「卸载时」精确配对使用。
4. 进 Location：确保 Gameplay 场景**已加载**（没有则加载）；回 Menu：确保 Gameplay 场景**已卸载**（加载了则卸掉）。搞反 → 主菜单残留玩法管理器（多余系统在跑/重复 spawn），或 Location 里没有 spawn 系统（玩家无法生成）。
5. 多个订阅者都会执行加载逻辑 → 同一场景被加载多次 / 状态机冲突。约束不强制是因为用的是裸 `UnityAction` 字段（多播委托天然允许多订阅），靠团队约定与字段 Header 注释维持。
6. 缺少常驻 Gameplay 管理器场景（spawn 系统等）。表现：玩家不生成、相机无目标、依赖管理器的系统全部 NullReference。冷启动路径同步加载 Gameplay 场景来补齐。
7. Entrance 的 Awake 晚于广播 → 它订阅时事件已过去，永远收不到，相机转场不触发（停在入口镜头或无转场）。对「可能错过的一次性通知」，RuntimeAnchor（持久 + 可补触发）比纯瞬时事件更稳；或保证 SceneLoader 广播在所有场景对象 Awake 之后（如用 Start 或延迟一帧）。
</details>

---

## ✍️ 实操题（3）

1. 当前加载链路无超时保护。设计一个「加载超过 N 秒则报错并复位 `_isLoading`」的方案，说明放在协程/async 的哪个位置。
2. 给 `SceneLoader` 增加「加载进度」上报（0~1），经一个 `FloatEventChannelSO` 广播给 loading 界面。指出你会在哪里读取 `AsyncOperationHandle.PercentComplete` 并节流上报。
3. 现在「场景切换即存档」由 SaveSystem 订阅 `_loadLocation` 实现。若要支持「某些过场场景不存档」，在不破坏事件解耦的前提下，你会怎么改 `GameSceneSO` 或事件载荷？

<details><summary>参考答案要点</summary>

1. 在加载新场景的 `await`/协程等待处加超时：用 `Task.WhenAny(loadTask, Task.Delay(N))` 或协程里累计 `WaitForSeconds`；超时分支记录错误、卸载半成品、在 `finally` 复位 `_isLoading`，并广播一个失败事件供 UI 提示。
2. 在 `LoadNewScene` 后的协程里轮询 `_loadingOperationHandle.PercentComplete`，每隔 0.1s 或每变化 >0.05 才 RaiseEvent（节流），完成时强制上报 1。
3. 在 `GameSceneSO` 加 `bool saveOnLoad`（或新增场景类型），SaveSystem 的 `CacheLoadLocations` 里读 `locationSO.saveOnLoad` 决定是否写盘。不改事件签名即不破坏解耦——元数据随 SO 一起传递。
</details>
