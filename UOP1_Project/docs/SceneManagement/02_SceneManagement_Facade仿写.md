# SceneManagement Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `SceneLoader` 场景加载器 | 保留核心逻辑 | ✅保留 |
| `LocationExit` 场景出口 | 保留 | ✅保留 |
| `LocationEntrance` 场景入口 | 保留核心逻辑 | ✅保留 |
| `GameSceneSO` 场景配置 | 保留 | ✅保留 |
|` 路径存储 | 保留 | ✅保留 |
| Addressables加载 | 简化为SceneManager | ⚠️ 简化 |
| 淡入淡出 | 简化 | ⚠️ 简化 |

---

## 最小复刻代码（~100行）

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.SceneManagement;

// ==================== 场景配置 ====================

public enum GameSceneType
{
    Location,
    Menu,
    PersistentManagers
}

[CreateAssetMenu(menuName = "Scene Management/Game Scene")]
public class GameSceneSO : ScriptableObject
{
    public GameSceneType sceneType;
    public string sceneName;
}

// ==================== 路径存储 ====================

[CreateAssetMenu(menuName = "Scene Management/Path Storage")]
public class PathStorageSO : ScriptableObject
{
    public string lastPathTaken;
}

// ==================== 场景出口 ====================

public class LocationExit : MonoBehaviour
{
    [SerializeField] private GameSceneSO _locationToLoad;
    [SerializeField] private string _pathName;
    [SerializeField] private PathStorageSO _pathStorage;
    
    public UnityEvent<GameSceneSO> OnLoadRequested;
    
    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Player"))
        {
            _pathStorage.lastPathTaken = _pathName;
            OnLoadRequested?.Invoke(_locationToLoad);
        }
    }
}

// ==================== 场景加载器 ====================

public class SceneLoader : MonoBehaviour
{
    [SerializeField] private PathStorageSO _pathStorage;
    
    private bool _isLoading;
    
    private void OnEnable()
    {
        // 订阅加载请求
    }
    
    public void LoadScene(GameSceneSO scene)
    {
        if (_isLoading) return;
        StartCoroutine(LoadSceneCoroutine(scene));
    }
    
    private IEnumerator LoadSceneCoroutine(GameSceneSO scene)
    {
        _isLoading = true;
        
        // 卸载当前场景
        var currentScene = SceneManager.GetActiveScene();
        yield return SceneManager.UnloadSceneAsync(currentScene);
        
        // 加载新场景
        yield return SceneManager.LoadSceneAsync(scene.sceneName, LoadSceneMode.Additive);
        
        // 通知场景就绪
        _isLoading = false;
    }
}

// ==================== 使用示例 ====================

public class SpawnSystem : MonoBehaviour
{
    [SerializeField] private GameObject _playerPrefab;
    [SerializeField] private PathStorageSO _pathStorage;
    
    public void SpawnPlayer()
    {
        // 根据路径选择出生点
        var spawnPoint = GetSpawnPoint(_pathStorage.lastPathTaken);
        Instantiate(_playerPrefab, spawnPoint.position, Quaternion.identity);
    }
    
    private Transform GetSpawnPoint(string path)
    {
        // 查找匹配的入口
        var entrances = FindObjectsOfType<LocationEntrance>();
        foreach (var entrance in entrances)
        {
            if (entrance.pathName == path) return entrance.transform;
        }
        return GetDefaultSpawnPoint();
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 加载状态锁 | ✅保留 | 核心机制 |
| 路径系统 | ✅保留 | 核心机制 |
| 持久化管理器 | ✅保留 | 核心机制 |
| Addressables | ❌砍掉 | 平台细节 |
| **最容易搞错** | ⚠️ | 忘记设置 `_isLoading` 导致重复加载 |
