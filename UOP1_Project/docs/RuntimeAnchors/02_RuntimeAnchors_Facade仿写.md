# RuntimeAnchors Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `RuntimeAnchorBase<T>` 泛型基类 | 保留 | ✅保留 |
| `TransformAnchor` 特化 | 保留 | ✅保留 |
| `isSet` 状态标志 | 保留 | ✅保留 |
| `OnAnchorProvided` 事件 | 保留 | ✅保留 |
| `OnDisable` 清理 | 保留 | ✅保留 |
| `PathStorageSO` | 砍掉（业务特定） | ❌砍掉 |

---

## 最小复刻代码（~60行）

```csharp
using System;
using UnityEngine;

// ==================== 锚点基类 ====================

public class RuntimeAnchorBase<T> : ScriptableObject where T : UnityEngine.Object
{
    public event Action OnAnchorProvided;
    
    [SerializeField, ReadOnly] private T _value;
    [SerializeField, ReadOnly] private bool _isSet;
    
    public bool isSet => _isSet;
    public T Value => _isSet ? _value : null;

    public void Provide(T value)
    {
        if (value == null)
        {
            Debug.LogError($"Null value provided to {name}");
            return;
        }
        _value = value;
        _isSet = true;
        OnAnchorProvided?.Invoke();
    }

    public void Unset()
    {
        _value = null;
        _isSet = false;
    }

    protected virtual void OnDisable() => Unset();
}

// ==================== Transform 专用 ====================

[CreateAssetMenu(menuName = "Runtime Anchors/Transform")]
public class TransformAnchor : RuntimeAnchorBase<Transform> { }

// ==================== 使用示例 ====================

// 生产者
public class SpawnSystem : MonoBehaviour
{
    [SerializeField] private TransformAnchor _playerAnchor;
    
    public void SpawnPlayer()
    {
        var player = Instantiate(playerPrefab);
        _playerAnchor.Provide(player.transform);
    }
}

// 消费者
public class CameraManager : MonoBehaviour
{
    [SerializeField] private TransformAnchor _playerAnchor;
    
    private void OnEnable()
    {
        _playerAnchor.OnAnchorProvided += SetupCamera;
        if (_playerAnchor.isSet) SetupCamera();
    }
    
    private void OnDisable() => _playerAnchor.OnAnchorProvided -= SetupCamera;
    
    private void SetupCamera()
    {
        // 设置相机跟随目标
        followTarget = _playerAnchor.Value;
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| SO 存储运行时引用 | ✅保留 | 核心机制 |
| isSet 状态标志 | ✅保留 | 安全检查 |
| OnDisable 清理 | ✅保留 | 防止悬空引用 |
| **最容易搞错** | ⚠️ | 消费者未检查 isSet 直接访问 Value |
