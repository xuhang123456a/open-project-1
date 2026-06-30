# SaveSystem 模块考题

---

## 🟢 概念题（5）

1. 存档里为什么存 ItemSO 的 GUID，而不是它的名字或对象引用？
2. `Save` 类的角色是什么？为什么它的字段都是 public？
3. `SerializableScriptableObject` 在编辑器里靠什么把 GUID 写入 `_guid`？运行时还会变吗？
4. `FileManager` 的三个方法为什么都返回 bool 而不抛异常？
5. `SaveSystem` 为什么也是 ScriptableObject？

<details><summary>参考答案要点</summary>

1. 对象引用/指针无法跨会话存储；名字不稳定（可重名/改名）。GUID 是编辑期固化、跨会话跨打包稳定的唯一键，加载时经 Addressables 反查回真实 SO。
2. 纯数据 DTO（存档格式）。字段 public 是为避免写大量 trivial getter/setter（源码注释明说），且 `JsonUtility` 序列化字段更方便。
3. `OnValidate` 里 `AssetDatabase.AssetPathToGUID(GetAssetPath(this))`。运行时 GUID 已固化进序列化字段，不再变（`#if UNITY_EDITOR` 包裹，运行时不执行）。
4. 文件 IO 易失败（权限/磁盘满/路径错），用 bool + LogError 让上层优雅处理而非崩溃；存档失败不应让游戏挂掉。
5. 与 Pool/Events 同构——作为可被多系统 `[SerializeField]` 引用的全局存档服务，共享同一份 `saveData`。
</details>

---

## 🟡 机制题（6）

1. `SaveDataToDisk` 在写 JSON 前对 `_itemStacks` 做了什么？为什么必须先 Clear？
2. `LoadFromJson` 用 `JsonUtility.FromJsonOverwrite` 而非 `FromJson`，区别是什么？
3. `SaveDataToDisk` 里 `MoveFile(saveFilename, backupSaveFilename)` 的作用？它在写新档之前还是之后？
4. `LoadSavedInventory` 为什么是协程（IEnumerator）而不是普通方法？
5. `LoadFromFile` 在文件不存在时做了什么？这避免了什么后续问题？
6. `CacheLoadLocations` 把什么写进了 saveData？它由哪个事件触发？意味着什么存档策略？

<details><summary>参考答案要点</summary>

1. 先 `Clear()` 再把运行时 `_playerInventory.Items` 逐个转成 `SerializedItemStack`(Guid+Amount) 重填。必须 Clear 防止上次的旧条目残留导致重复/陈旧数据。
2. `FromJsonOverwrite` 把 JSON 反序列化**就地覆写到现有对象实例**（保留对象引用、未出现的字段保持原值）；`FromJson` 新建实例。前者适合覆写已存在的 `saveData`。
3. 把当前存档移为 `.bak` 备份。在写新档**之前**——若写新档崩溃，旧档以 .bak 形式幸存（写时备份容错）。
4. 每个 ItemSO 经 `Addressables.LoadAssetAsync` 异步加载（可能来自 AssetBundle/远端），需 `yield return` 等待完成，故必须协程。
5. 先 `File.WriteAllText(fullPath, "")` 创建空文件，再读。避免后续读取不存在文件失败，保证「首次启动无存档」也能正常走读流程（读到空串）。
6. 写入当前要加载的 Location 的 `Guid`（`saveData._locationId`）。由 `_loadLocation`(LoadEventChannelSO) 触发。策略：每次切换场景即自动存档当前位置（检查点式存档）。
</details>

---

## 🔴 架构陷阱题（7）

1. 【对称性漏洞】团队新增了 Buff 系统，在游戏里能用，但开发者只在 `Save` 加了字段、忘了在 `SaveDataToDisk` 写入。玩家存档读档后 Buff 会怎样？为什么不报错？
2. 【GUID 稳定性】某美术把 `Apple.asset` 删了重建了一个同名同内容的资产。旧存档里该物品的 GUID 还能解析吗？表现是什么？这暴露 GUID 方案的什么前提？
3. 【覆写灾难】若把 `SaveDataToDisk` 改成直接 `WriteToFile(saveFilename, json)`（去掉 MoveFile 备份）。在写入过程中断电会怎样？为什么「写时备份」比它安全？
4. 【同底座差异】SaveSystem 与 AudioManager 都订阅事件通道做被动响应，但 SaveSystem 还被 StartGame 主动调用方法。对比「事件驱动」与「直接方法调用」两种交互在本模块的分工。
5. 【异步顺序】`StartGame.LoadSaveGame` 里先 `yield return StartCoroutine(LoadSavedInventory())` 再加载 Location。若把两者并行（不等背包加载完就加载场景），可能出现什么 bug？
6. 【FromJsonOverwrite 陷阱】`saveData` 是 SO 上的字段（跨 Play 会话可能残留上次数据）。连续「继续游戏」两次但第二次存档文件为空，`FromJsonOverwrite("")` 会怎样？saveData 里会残留什么？
7. 【双消费者正交】SceneLoader 和 SaveSystem 都订阅 `_loadLocation`。若 SaveSystem 的 `CacheLoadLocations` 抛异常，会影响 SceneLoader 的加载吗？多播委托异常传播在这里有何风险？

<details><summary>参考答案要点</summary>

1. Buff 状态不会被写盘（DTO 里始终是默认值），读档后 Buff 全部丢失/为默认。不报错因为序列化只处理 DTO 里实际写入的字段，漏写是「静默丢失」——这正是「保存/加载字段集必须对称」的反例。
2. 不能解析（GUID 变了）。`Addressables.LoadAssetAsync(旧guid)` 失败，该物品在读档后丢失（源码里加载失败分支不 Add）。前提：GUID 必须在资产整个生命周期稳定，删除重建会生成新 GUID——这是 GUID 引用持久化的脆弱点。
3. 唯一存档文件在写入途中被截断 → 永久损坏，玩家进度全失且无备份。写时备份先把旧档移为 .bak，新档写坏了至少能从 .bak 恢复——保证任何时刻都有一份完好存档。
4. 事件驱动用于「被动响应游戏中自然发生的事」（切场景→存位置、改设置→存设置），解耦且不需调用方知道 SaveSystem；直接方法调用用于「玩家主动的存档生命周期操作」（新游戏/继续游戏的明确编排），需要顺序控制与返回值（如 `LoadSaveDataFromDisk` 返回是否有存档）。
5. 场景加载完触发 spawn/UI，但背包可能还没反水合完——玩家进场景时背包为空或半空，UI 显示错误，依赖背包的逻辑（如任务检查）误判。串行 yield 保证「数据就绪 → 再进场景」的顺序。
6. `FromJsonOverwrite("")` 对空串通常不覆写任何字段（保持 saveData 现有值）——于是残留上一次会话/上一次继续游戏的旧数据，玩家可能加载到错误的旧进度。需对空串特判（如视为新档）。
7. 取决于订阅顺序与异常传播。多播委托中某订阅者抛异常会中断后续订阅者调用。若 SaveSystem 注册在 SceneLoader 之前且抛异常，SceneLoader 可能收不到该次事件 → 场景不加载。风险：一个消费者的失败拖垮同通道其他正交消费者。应在各回调内部 try-catch 隔离。
</details>

---

## ✍️ 实操题（3）

1. 把当前「先备份再写」升级为更安全的「写临时文件 → 成功后原子 rename 覆盖正式档 → 再留一份 .bak」。写出关键步骤顺序，并说明它比原版多保证了什么。
2. 设计一个「存档版本号 + 迁移」机制：当 `Save` 结构升级（如重命名字段）时，旧存档仍能被读取并迁移。指出版本号存哪、迁移在读档流程的哪一步做。
3. `RehydrateInventory` 中若某 GUID 解析失败当前是「丢弃该物品」。改成「记录丢失清单并继续」，并在加载完成后把丢失清单经一个事件通道上报。写出关键改动点。

<details><summary>参考答案要点</summary>

1. ①写到 `save.tmp`；②（可选）读回校验 JSON 合法；③把现有 `save.chop` 移为 `save.chop.bak`；④把 `save.tmp` 原子 rename 为 `save.chop`。比原版多保证：新档在 rename 前已完整写好并校验，rename 是原子操作——任何中断点要么是旧档完好、要么是新档完整，绝不出现半截正式档。
2. 在 `Save` 加 `int _version` 字段，落盘时写当前版本。读档 `LoadFromJson` 后、反水合前，比较 `_version` 与当前代码版本，按差值依次执行迁移函数（v1→v2→v3）改写 DTO 字段，再继续反水合。
3. `RehydrateInventory` 里把 `item==null` 分支改为 `lostGuids.Add(ss.ItemGuid)`（不再静默丢弃）；遍历结束后若 `lostGuids` 非空，`_itemLostChannel.RaiseEvent(lostGuids)`（新增一个事件通道字段）。UI 订阅该通道提示玩家「部分物品因资源缺失丢失」。
</details>
