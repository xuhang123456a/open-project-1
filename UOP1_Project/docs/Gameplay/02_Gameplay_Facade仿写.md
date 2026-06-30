# Gameplay Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `GameState` 枚举 | 保留 | ✅保留 |
| `GameStateSO` 状态管理 | 保留核心逻辑 | ✅保留 |
| `GameManager` 启动逻辑 | 简化 | ⚠️ 简化 |
| `SpawnSystem` 生成逻辑 | 保留核心逻辑 | ✅保留 |
| 战斗状态事件广播 | 保留 | ✅保留 |
| 状态回退 | 保留 | ✅保留 |

---

## 最小复刻代码（~140行）

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

// ==================== 游戏状态枚举 ====================

public enum GameState
{
    Gameplay,
    Pause,
    Inventory,
    Dialogue,
    Cutscene,
    LocationTransition,
    Combat
}

// ==================== 游戏状态管理 ====================

public class GameStateSO : ScriptableObject
{
    [SerializeField] private GameState _currentGameState = GameState.Gameplay;
    [SerializeField] private GameState _previousGameState = GameState.Gameplay;
    
    public GameState CurrentGameState => _currentGameState;
    public GameState PreviousGameState => _previousGameState;
    
    public event System.Action<GameState, GameState> OnStateChanged;
    public event System.Action<bool> OnCombatStateChanged;
    
    private readonly List<Transform> _alertEnemies = new List<Transform>();

    public void AddAlertEnemy(Transform enemy)
    {
        if (!_alertEnemies.Contains(enemy))
        {
            _alertEnemies.Add(enemy);
            UpdateGameState(GameState.Combat);
        }
    }

    public void RemoveAlertEnemy(Transform enemy)
    {
        if (_alertEnemies.Remove(enemy) && _alertEnemies.Count == 0)
        {
            UpdateGameState(GameState.Gameplay);
        }
    }

    public void UpdateGameState(GameState newState)
    {
        if (newState == _currentGameState) return;
        
        _previousGameState = _currentGameState;
        _currentGameState = newState;
        
        // 广播战斗状态变化
        if (newState == GameState.Combat)
            OnCombatStateChanged?.Invoke(true);
        else if (_previousGameState == GameState.Combat)
            OnCombatStateChanged?.Invoke(false);
        
        OnStateChanged?.Invoke(_previousGameState, _currentGameState);
    }

    public void ResetToPreviousGameState()
    {
        if (_previousGameState == _currentGameState) return;
        
        var temp = _currentGameState;
        UpdateGameState(_previousGameState);
        _previousGameState = temp;
    }
}

// ==================== 玩家生成系统 ====================

public class SpawnSystem : MonoBehaviour
{
    [SerializeField] private GameObject _playerPrefab;
    [SerializeField] private TransformAnchor _playerTransformAnchor;
    [SerializeField] private PathStorageSO _pathStorage;
    
    private Transform[] _spawnPoints;
    private Transform _defaultSpawnPoint;

    private void Awake()
    {
        _spawnPoints = FindObjectsOfType<LocationEntrance>()
            .Select(e => e.transform).ToArray();
        _defaultSpawnPoint = transform.GetChild(0);
    }

    public void SpawnPlayer()
    {
        var spawnPoint = GetSpawnPoint();
        var player = Instantiate(_playerPrefab, spawnPoint.position, spawnPoint.rotation);
        _playerTransformAnchor.Provide(player.transform);
    }

    private Transform GetSpawnPoint()
    {
        if (_pathStorage?.lastPathTaken == null) return _defaultSpawnPoint;
        
        foreach (var point in _spawnPoints)
        {
            var entrance = point.GetComponent<LocationEntrance>();
            if (entrance != null && entrance.EntrancePath == _pathStorage.lastPathTaken)
                return point;
        }
        
        return _defaultSpawnPoint;
    }
}

// ==================== 使用示例 ====================

public class Enemy : MonoBehaviour
{
    [SerializeField] private GameStateSO _gameState;
    
    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player"))
            _gameState.AddAlertEnemy(transform);
    }
    
    private void OnTriggerExit(Collider other)
    {
        if (other.CompareTag("Player"))
            _gameState.RemoveAlertEnemy(transform);
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 状态枚举 | ✅保留 | 核心机制 |
| 状态切换广播 | ✅保留 | 核心机制 |
| 战斗状态管理 | ✅保留 | 核心机制 |
| 状态回退 | ✅保留 | 核心机制 |
| 生成点选择 | ✅保留 | 核心机制 |
| **最容易搞错** | ⚠️ | 忘记在状态切换时广播事件 |
