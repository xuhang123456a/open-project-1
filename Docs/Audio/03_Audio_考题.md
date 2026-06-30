# Audio 模块考题

---

## 🟢 概念题（5）

1. `AudioManager` 自己不持有任何声音逻辑，它的核心职责是哪几步编排？
2. 一次 `PlayAudioCue` 为什么可能借出多个 `SoundEmitter`？它们共用几个 `AudioCueKey`？
3. `AudioCueKey` 为什么是 `struct` 而不是 `class`？它的身份由什么构成？
4. `SoundEmitterVault` 为什么不用 `Dictionary<AudioCueKey, SoundEmitter[]>` 而用两条并行 `List`？代价是什么？
5. Audio 模块同时消费了哪三个核心底座？各负责什么？

<details><summary>参考答案要点</summary>

1. 监听 cue 通道 → 从池借 emitter → 让 emitter 播放 → 把 (key→emitters) 登记到金库 → 在结束/停止时归还 emitter（并应移除 key）。
2. 一个 AudioCue 可含多个并行 clip（`GetClips()` 返回数组），每个 clip 借一个 emitter。整组共用**一个** key。
3. 它是轻量不可变句柄，值传递无堆分配；身份 = 自增序号 `Value` + `AudioCueSO` 引用，二者共同唯一（重写了 ==/Equals/GetHashCode）。
4. 有意的简单实现（声音并发量小）。代价：`Get`/`Remove` 用 `FindIndex` 线性查找，O(n)；且两列表索引必须始终同步。
5. Pool（SoundEmitter 借还复用）、Factory（SoundEmitterFactorySO `Instantiate` 创建）、Events（AudioCueEventChannelSO 带返回值请求通道）。
</details>

---

## 🟡 机制题（6）

1. 非循环 SFX 是怎么知道「自己播完了」的？循环音呢？
2. `AudioManager.OnEnable` 订阅、`OnDestroy` 退订。为什么这里用 `OnDestroy` 而不像 Listener 用 `OnDisable`？
3. `PlayMusicTrack` 切到一首正在播放的相同歌曲时返回什么？为什么这样设计？
4. `Finish()` 和 `Stop()` 对一个循环音的区别是什么？
5. `AudioClipsGroup.GetNextClip` 的 `RandomNoImmediateRepeat` 用 do-while 实现了什么？只有一个 clip 时会怎样？
6. emitter 复用时 `PlayAudioClip` 里 `_audioSource.time = 0f` 这行注释说明它防什么问题？

<details><summary>参考答案要点</summary>

1. 非循环音在 `PlayAudioClip` 里启动 `FinishedPlaying(clip.length)` 协程，到点 `NotifyBeingDone` 触发 `OnSoundFinishedPlaying`。循环音不启动协程，只能靠外部 `Stop`/`Finish`。
2. AudioManager 通常常驻（管理器场景），其生命周期等同对象存在期；用 OnDestroy 保证只在真正销毁时退订，而非每次失活。Listener 是场景物体，需随激活态订阅退订。
3. 返回 `AudioCueKey.Invalid`（且不重新播放）。避免同一首歌重复淡入造成叠音/打断自己。
4. `Stop` 立即停止并触发归还；`Finish` 只把 `loop=false` 并按剩余时长启动协程，让当前这遍自然播完再结束（优雅收尾）。
5. 保证本次随机出的 clip 不等于上一次播放的 clip（避免听感上连续重复）。只有 1 个 clip 时函数开头 fast-out 直接返回它（不进随机分支，否则 do-while 死循环）。
6. 防 emitter 被复用时残留上一次（可能是长音乐）的播放进度——重置到 0 避免短 SFX 从中间开始播。
</details>

---

## 🔴 架构陷阱题（7）

1. 【源码 TODO】`StopAndCleanEmitter` 在「自然播完」后归还了 emitter，却**没从金库 Remove key**。描述这个悬空 key 会引发的具体 bug 序列（提示：emitter 被池复用后）。
2. 【漏步后果】一个循环 SFX（如脚步环境音）的请求方忘记调 `StopAudioCue`。随着时间推移池会怎样？最终表现是什么？
3. 【同底座差异】对比 `SoundEmitterVault`（key↔资源映射）与 StateMachine 的 `createdInstances`（SO↔运行时实例映射）：两者都是「映射表」，但生命周期和去重目的有何根本不同？
4. 【句柄失效】`AudioCue.cs` 持有 `controlKey`。它所属的 SFX 自然播完后（按修复版）key 已从金库移除，但 `AudioCue` 仍持有旧 `controlKey`。之后调 `StopAudioCue` 会怎样？源码靠什么避免误操作？
5. 【相等性陷阱】`AudioCueKey.GetHashCode` 用 `Value ^ AudioCue.GetHashCode()`。若某 cue 引用为 null（如 Invalid），调 `GetHashCode` 会怎样？这对把 key 放进 Dictionary 有何风险？
6. 【并发竞态】快速连续两次切音乐：第一次 `FadeMusicOut` 的 DOTween 补间未完成，第二次又来切。`_musicSoundEmitter` 单字段会怎样？哪个 emitter 可能不被归还？
7. 【借还时机分散】emitter 的归还散布在 `StopAndCleanEmitter`/`StopMusicEmitter`/`OnSoundEmitterFinishedPlaying` 多处。这种「多入口归还」相比「单入口归还」最大的风险是什么？

<details><summary>参考答案要点</summary>

1. emitter 归还入池 → 后续别的播放请求 `Request` 到同一 emitter 并登记新 key → 此时金库里仍有旧 key 指向这个 emitter（现属新播放）。若有人持旧 key 调 `Stop`，`Get` 命中旧 key 找到该 emitter，把**正在为新请求服务的 emitter** 停掉并归还——误杀正在播放的声音 + 可能 double-return。
2. 循环音不会自动归还，该 emitter 永久脱离池（Leaked）。每泄漏一个，池可用数减一；长期运行池被掏空，新的播放请求 `Request` 不断 `Create` 新 emitter，实例膨胀，且老的循环音永远在响。
3. 金库是**运行期动态**映射（随每次播放 Add、结束 Remove，生命周期短、条目不断增删），目的是「凭外部句柄反查内部资源」；`createdInstances` 是**构建期一次性**映射（Awake 建图时填充，之后只读），目的是「同一定义全图只实例化一份」（去重）。一个是动态注册表，一个是构建期去重缓存。
4. 修复版：`Get(旧key)` 返回 false（已移除），`StopAudioCue` 返回 false，`AudioCue` 据此把 `controlKey = Invalid`。源码 `StopAudioCue`/`FinishAudioCue` 通过返回值 false 让请求方复位 controlKey，避免反复用失效 key——但前提是金库已正确移除（原版未移除时这个保护失效）。
5. `null.GetHashCode()` 抛 NullReferenceException。Invalid 的 cue 为 null，若把 Invalid 当 Dictionary key 会崩。原版用并行 List + `==`（`==` 里比较引用安全），规避了对 null cue 求哈希——这也是不用 Dictionary 的隐性原因之一。
6. 第一次的旧 emitter 引用被第二次切歌覆盖（`_musicSoundEmitter = _pool.Request()`），其 DOFade 完成回调仍指向旧 emitter 但 manager 已不再引用它；旧 emitter 可能既不被 `StopMusicEmitter` 正常归还（时序错乱），造成泄漏或两个音乐 emitter 同时存在。
7. 多入口归还容易出现「某条路径漏归还（泄漏）」或「两条路径都归还同一 emitter（double-return，池中重复）」。单入口归还（所有结束路径汇聚到一个 Return 点 + 幂等校验）更易保证「借一次还一次」不变量。
</details>

---

## ✍️ 实操题（3）

1. 修复源码 TODO：让「自然播完」路径也从金库移除对应 key。给出你会在 `SoundEmitterVault` 增加的方法签名，以及 `AudioManager` 的改动点。
2. 把 `SoundEmitterVault` 从并行 List 重构为 `Dictionary<AudioCueKey, SoundEmitter[]>`。指出你必须先解决 `AudioCueKey` 的哪个问题（提示：哈希 + null cue），否则 Invalid key 会崩。
3. 设计一个「emitter 借还幂等 + 泄漏检测」机制：当某 emitter 被归还两次或借出后超过 T 秒未归还时打日志。说明你会在 Pool 还是 AudioManager 层加这层逻辑、为什么。

<details><summary>参考答案要点</summary>

1. `Vault` 加 `bool RemoveByEmitter(SoundEmitter e, out AudioCueKey key)`（遍历 `_emittersList`，`Array.IndexOf` 命中即 RemoveAt 两列表）。`AudioManager.OnSoundEmitterFinishedPlaying`/`StopAndCleanEmitter` 在 `pool.Return` 前调用它清 key。
2. 先修 `GetHashCode`：当 `AudioCue == null` 时返回 `Value.GetHashCode()`（不对 null 求哈希），并确保 `Equals` 对 null cue 安全。否则把 Invalid 或 cue 为 null 的 key 作 Dictionary key 会 NRE。
3. 在 Pool 层加：因为「借出/归还」语义属于池，泄漏与 double-return 是池的不变量问题（与具体是不是 audio 无关）。实现：`HashSet` 记在用集合做归还幂等校验；借出时记时间戳，定期扫描超时未还者打日志。AudioManager 层只负责业务回收时机，不该重复造池级安全网。
</details>
