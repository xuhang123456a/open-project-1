# Inventory 模块考题

---

## 🟢 概念题（5）

1. `InventorySO` 用 `_items` 和 `_defaultItems` 两个列表，各代表什么？
2. `ItemStack` 为什么设计成 class 而不是 struct？`Item` 和 `Amount` 的可变性有何不同？
3. `Add` 在什么条件下会累加已有物品的数量？什么条件下即使物品已存在也不增量？
4. `IngredientsAvailability` 与 `hasIngredients` 两个方法的返回值和用途分别是什么？
5. `ItemSO` 继承自 `SerializableScriptableObject` 而非普通 ScriptableObject，多了什么能力？为谁服务？

<details><summary>参考答案要点</summary>

1. `_items` 是运行时实际背包内容；`_defaultItems` 是新游戏初始内容（配置数据），`Init` 据它填充背包。
2. class 让列表里取出的 ItemStack 是同一引用，直接改 `Amount` 即改背包（无需写回）。`Item` 只读（栈建立后物品不可换），`Amount` public 可变。
3. 找到同物品栈且 `ItemType.ActionType == Use` 时累加；非 Use 类型（如关键道具）找到也直接 return 不增量。
4. `IngredientsAvailability` 返回 `bool[]`（每个原料是否齐备，供 UI 逐项显示缺料）；`hasIngredients` 返回单 `bool`（整体能否合成）。
5. 多了稳定的 `Guid`。为 SaveSystem 服务——背包物品落盘时存 GUID，加载时经 Addressables 反查重建 ItemSO 引用。
</details>

---

## 🟡 机制题（6）

1. `Init` 里为什么用 `new ItemStack(item)` 而不是直接 `_items.Add(item)`（item 来自 `_defaultItems`）？
2. `Remove` 在 `for` 循环里调用 `_items.Remove(currentItemStack)` 后紧跟 `return`。这个 return 起到什么关键作用？
3. `Add` 找到同物品后无论是否累加都 `return`。如果去掉这个 return 会怎样？
4. `hasIngredients` 的 `!ingredients.Exists(j => !_items.Exists(...))` 用了双重否定。把它翻译成正向逻辑是什么？
5. `Count` 返回 0 有两种含义，分别是什么？调用方如何区分？
6. `Contains`/`Count`/`Add`/`Remove` 都用 `==` 比较 `ItemSO`。这里比较的是引用还是值？两个内容相同但不同资产的 ItemSO 会被视为相等吗？

<details><summary>参考答案要点</summary>

1. 防污染默认列表：ItemStack 是 class，直接 Add 会让 `_items` 与 `_defaultItems` 共享同一对象，运行时改 Amount 会改写默认配置；深拷贝隔离两者。
2. 删除元素后立即退出循环，避免继续用失效的索引 `i` 访问已被修改的列表（迭代安全）。因为每次 Remove 只处理一个物品，命中即完成。
3. 累加分支会继续遍历后续元素；若背包存在多个同物品栈（理论上不该有），可能重复处理；更主要是逻辑冗余且若后面再匹配会二次累加。原设计假设每物品唯一栈，return 保证只处理首个命中。
4. 正向：「对所有原料 j，背包都存在某栈 o 满足 o.Item==j.Item 且 o.Amount>=j.Amount」即「每种原料都备足」。
5. ①背包没有该物品（不 Contains）；②有该物品栈但 Amount 恰为 0（边界，正常会被 Remove 清掉，但理论存在）。区分：先用 `Contains` 判断存在性，再用 `Count` 取量。
6. 比较引用（ScriptableObject 未重写 ==，用默认引用相等）。内容相同但不同资产实例**不相等**——所以同一物品必须全局引用同一份 ItemSO 资产。
</details>

---

## 🔴 架构陷阱题（7）

1. 【污染陷阱】某开发者把 `Init` 改成 `_items = new List<ItemStack>(_defaultItems)`（浅拷贝列表）。玩家玩一局消耗了初始苹果，再开新游戏。会发生什么？为什么？
2. 【迭代失效】产品要加一个 `RemoveAll(List<Item> items)` 批量移除。新手直接 `foreach (var it in items) Remove(it)` 看似没事，但若改成「在一个 for 遍历 `_items` 中边遍历边移除多个不同物品」会怎样？正确写法？
3. 【堆叠规则隐藏】策划反馈「Boss 钥匙能叠到 5 把」。从 `Add` 源码角度，最可能是哪里配置错了？这条规则藏在代码哪个判断里？
4. 【同底座差异】`InventorySO` 与 `AudioManager` 都是「共享 SO + 持有可变集合」。但 Inventory 的集合是业务数据、Audio 的金库是资源句柄表。两者在「集合元素生命周期」上有何根本不同？
5. 【SO 状态残留】`InventorySO._items` 是 SO 资产字段。编辑器里反复进出 Play、玩到一半退出，`_items` 可能残留上次会话的内容。下次 Play 若没调 `Init`/读档会怎样？这与 Pool 的 `OnDisable` 复位是同类问题吗？
6. 【引用相等假设】美术复制了一份 `Apple.asset` 叫 `Apple2.asset`（内容一样）。场景里有的拾取物用 Apple、有的用 Apple2。背包 `Count(Apple)` 会把 Apple2 算进去吗？UI 会怎样表现？
7. 【并发修改】UI 正在 `foreach (var s in inventory.Items)` 渲染列表时，一个拾取事件触发 `Add` 往 `_items` 里加了新栈。会发生什么？`Items` 暴露为可枚举的设计有何风险？

<details><summary>参考答案要点</summary>

1. 浅拷贝列表共享同一批 ItemStack 对象引用。玩第一局消耗苹果（改了共享 ItemStack 的 Amount，或 Remove 把它从 `_defaultItems` 也移除了——取决于具体写法），新游戏 Init 拿到被污染的默认值（苹果变少或消失）。根因：列表浅拷贝复制的是引用，元素仍共享。
2. 在单个 for 遍历 `_items` 中移除多个元素：移除后 `Count` 减小而 `i` 继续递增，会跳过元素或越界。正确：反向 for（`for i=Count-1..0`）、`_items.RemoveAll(s => items.Contains(s.Item))`、或对每个目标物品独立调用现有 `Remove`（各自 return 安全）。
3. Boss 钥匙的 `ItemTypeSO.ActionType` 被错配成了 `Use`（而非 KeyItem）。规则藏在 `Add` 的 `if (currentItemStack.Item.ItemType.ActionType == ItemInventoryActionType.Use)` 判断里——只有 Use 才累加。
4. Inventory 元素（ItemStack）随业务存活，是玩家长期持有的状态数据，生命周期 = 拥有期；Audio 金库元素（key→emitter[]）是短生命周期的播放会话，播完即应移除，元素不断增删。前者是持久状态集，后者是瞬态注册表。
5. 背包会残留上次会话内容（脏数据），玩家进新游戏却带着旧背包。是同类问题——SO 资产字段跨 Play 残留，需显式重置（Inventory 靠 `Init`/读档，Pool 靠 `OnDisable` 清空）。SO 的「资产即持久」是双刃剑：共享方便但必须主动复位。
6. 不会。`==` 比引用，Apple 与 Apple2 是不同资产实例，`Count(Apple)` 只数 Apple 栈。UI 会出现「两种看起来一样的苹果分两栈显示」，玩家困惑。根因：同一逻辑物品必须全局唯一资产引用。
7. 若 `Add` 在 UI 的 foreach 期间修改 `_items`，会抛 `InvalidOperationException`（集合被修改）。`Items` 直接暴露内部 List 引用让外部能在遍历期被异步改动。风险规避：暴露只读快照副本、或用 `IReadOnlyList` + 约定不在遍历中改、或事件驱动让 UI 在数据稳定后重建。
</details>

---

## ✍️ 实操题（3）

1. 当前 `Items` 直接暴露内部 `List<ItemStack>`。改造为「外部只读、修改只能经 Add/Remove」，并保证 UI 遍历时不会因并发 Add 崩溃。给出关键改动。
2. 实现一个 `ConsumeIngredients(List<ItemStack> recipe)`：先用 `hasIngredients` 校验，齐备则批量扣减原料、返回 true，否则不动任何数据返回 false。注意原子性（要么全扣要么不扣）。
3. 给 `Add` 增加「堆叠上限」支持（如可消耗品最多 99）：超过上限时多余的不入栈并返回实际加入量。说明你会如何不破坏「非 Use 类型不堆叠」的现有规则。

<details><summary>参考答案要点</summary>

1. 把 `Items` 改为返回 `IReadOnlyList<ItemStack>` 或 `_items.AsReadOnly()`；UI 渲染前取一份快照 `var snapshot = inventory.Items.ToList()` 再遍历，避免遍历期被 Add 修改原集合而抛异常。
2. 先 `if (!HasIngredients(recipe)) return false;`（校验在前，保证原子性）；通过后再 `foreach (var need in recipe) Remove(need.Item, need.Amount);` 返回 true。因为已校验齐备，扣减不会出现负数中断，满足「要么全扣要么不扣」。
3. 在 `Add` 的 Use 累加分支里：`int space = max - stack.Amount; int added = Min(count, space); stack.Amount += added; return added;`（满了则 added 可能 0）。非 Use 分支保持原样（命中直接 return，返回 0 或既有逻辑）——上限逻辑只加在 `ActionType==Use` 的 if 内部，不触碰关键道具分支。
</details>
