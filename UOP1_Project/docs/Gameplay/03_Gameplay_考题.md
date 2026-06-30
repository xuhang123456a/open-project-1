# Gameplay 考题

## 🟢 概念题

**Q1: `GameState` 枚举定义了哪些状态？**

<details>
<summary>参考答案</summary>

Gameplay、Pause、Inventory、Dialogue、Cutscene、LocationTransition、Combat。
</details>

---

**Q2: `GameStateSO` 的 `_previousGameState` 字段有什么作用？**

<details>
<summary>参考答案</summary>

支持状态回退（`ResetToPreviousGameState()`），例如关闭菜单后回到之前的状态。
</details>

---

**Q3: `AddAlertEnemy()` 和 `RemoveAlertEnemy()` 是如何管理战斗状态的？**

<details>
<summary>参考答案</summary>

- `AddAlertEnemy`：添加敌人到列表，切换到 Combat
- `RemoveAlertEnemy`：从列表移除，列表为空时回到 Gameplay
</details>

---

**Q4: `SpawnSystem` 如何选择生成点？**

<details>
<summary>参考答案</summary>

根据 `PathStorageSO.lastPathTaken` 查找匹配的 `LocationEntrance`，找不到则使用默认生成点。
</details>

---

**Q5: `GameManager.StartGame()` 做了什么？**

<details>
<summary>参考答案</summary>

初始化游戏状态为 Gameplay，启动任务系统。
</details>

---

## 🟡 机制题

**Q6: 如果同一个敌人被 `AddAlertEnemy()` 调用两次，会发生什么？**

<details>
<summary>参考答案</summary>

第二次调用被忽略（`!_alertEnemies.Contains(enemy)` 检查）。
</details>

---

**Q7: `ResetToPreviousGameState()` 是如何实现状态回退的？**

<details>
<summary>参考答案</summary>

交换 `_currentGameState` 和 `_previousGameState`，并广播状态变化事件。
</details>

---

**Q8: 如果 `PathStorageSO.lastPathTaken` 为 null，`SpawnSystem` 会使用哪个生成点？**

<details>
<summary>参考答案</summary>

默认生成点（`_defaultSpawnPoint`）。
</details>

---

**Q9: `UpdateGameState()` 中为什么要检查 `newState == _currentGameState`？**

<details>
<summary>参考答案</summary>

避免重复触发相同状态的变化事件。
</details>

---

**Q10: 战斗状态事件是在什么时候广播的？**

<details>
<summary>参考答案</summary>

- 进入 Combat：`OnCombatStateChanged?.Invoke(true)`
- 离开 Combat（从 Combat 切换到其他）：`OnCombatStateChanged?.Invoke(false)`
</details>

---

**Q11: `SpawnSystem` 为什么要通过 `TransformAnchor` 传递玩家引用？**

<details>
<summary>参考答案</summary>

解耦生成系统和消费系统（如 CameraManager），消费者通过锚点获取引用。
</details>

---

## � 架构陷阱题

**Q12: 【同底座差异】`GameStateSO` 和 StateMachine 的状态有什么区别？**

<details>
<summary>参考答案</summary>

- `GameStateSO`：全局游戏状态（暂停、对话等），影响整个游戏
- StateMachine：单个实体的状态（移动、攻击等），只影响该实体
</details>

---

**Q13: 【漏步后果】如果 `RemoveAlertEnemy()` 忘记检查列表为空，会有什么问题？**

<details>
<summary>参考答案</summary>

即使仍有敌人在警戒区，状态也会错误地切换回 Gameplay。
</details>

---

**Q14: 【架构陷阱】如果两个系统同时调用 `UpdateGameState()`，会有什么问题？**

<details>
<summary>参考答案</summary>

后调用的覆盖先调用的，可能导致状态不一致。需要状态机或锁机制。
</details>

---

**Q15: 【同底座差异】`GameStateSO` 和 `BoolEventChannelSO` 在战斗状态管理中如何配合？**

<details>
<summary>参考答案</summary>

- `GameStateSO`：管理状态逻辑
- `BoolEventChannelSO`：广播状态变化给 UI 和其他系统
</details>

---

**Q16: 【漏步后果】如果 `SpawnSystem` 在玩家生成前启用输入，会有什么问题？**

<details>
<summary>参考答案</summary>

玩家可能在世界未准备好时移动，导致掉落或卡墙。
</details>

---

**Q17: 【架构陷阱】`PathStorageSO` 和 `TransformAnchor` 的主要区别是什么？**

<details>
<summary>参考答案</summary>

- `PathStorageSO`：存储路径数据（值类型）
- `TransformAnchor`：存储 Transform 引用（引用类型）
</details>

---

**Q18: 【漏步后果】如果 `GameManager` 在 `StartGame()` 中忘记启动任务系统，会有什么问题？**

<details>
<summary>参考答案</summary>

任务系统不会初始化，玩家无法进行任务。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `PauseMenu`，在打开时切换到 Pause 状态，关闭时回退到之前的状态。**

<details>
<summary>参考答案</summary>

```csharp
public class PauseMenu : MonoBehaviour
{
    [SerializeField] private GameStateSO _gameState;
    
    public void OpenPause()
    {
        _gameState.UpdateGameState(GameState.Pause);
    }
    
    public void ClosePause()
    {
        _gameState.ResetToPreviousGameState();
    }
}
```
</details>

---

**Q20: 实现一个 `CombatUI`，监听战斗状态事件并显示/隐藏战斗指示器。**

<details>
<summary>参考答案</summary>

```csharp
public class CombatUI : MonoBehaviour
{
    [SerializeField] private GameObject _combatIndicator;
    [SerializeField] private BoolEventChannelSO _onCombatStateChanged;
    
    private void OnEnable() => _onCombatStateChanged.OnEventRaised += OnCombatChanged;
    private void OnDisable() => _onCombatStateChanged.OnEventRaised -= OnCombatChanged;
    
    private void OnCombatChanged(bool inCombat) => _combatIndicator.SetActive(inCombat);
}
```
</details>

---

**Q21: 假设有一个 Bug：关闭菜单后状态没有正确回退。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. `_previousGameState` 未正确更新
2. `ResetToPreviousGameState()` 未被调用
3. 状态切换时广播了错误的事件
</details>
