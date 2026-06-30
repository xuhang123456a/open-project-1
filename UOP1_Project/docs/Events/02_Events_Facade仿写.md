# Events Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| 多种类型 Channel（Void/Bool/Int/Float/GameObject等） | 泛型 `EventChannel<T>` | ⚠️ 简化 |
| `VoidEventListener` MonoBehaviour 桥接 | 保留核心逻辑 | ✅保留 |
| `UnityAction` / `UnityEvent` | 使用 C# `Action<T>` | ⚠️ 简化 |
| SO 配置 | 保留（核心机制） | ✅保留 |
| `AudioCueEventChannelSO` 特殊方法 | 砍掉（业务特定） | ❌砍掉 |

---

## 最小复刻代码（~120行）

```csharp
using System;
using UnityEngine;

// ==================== 事件通道 ====================

[CreateAssetMenu(menuName = "Events/Event Channel")]
public class EventChannelSO<T> : ScriptableObject
{
    public event Action<T> OnEventRaised;

    public void RaiseEvent(T value)
    {
        OnEventRaised?.Invoke(value);
    }
}

// 无参数版本特化
[CreateAssetMenu(menuName = "Events/Void Event Channel")]
public class VoidEventChannelSO : ScriptableObject
{
    public event Action OnEventRaised;

    public void RaiseEvent()
    {
        OnEventRaised?.Invoke();
    }
}

// ==================== MonoBehaviour 桥接 ====================

public class EventListener<T> : MonoBehaviour
{
    [SerializeField] private EventChannelSO<T> _channel;
    [SerializeField] private UnityEvent<T> _response;

    private void OnEnable()
    {
        if (_channel != null)
            _channel.OnEventRaised += Respond;
    }

    private void OnDisable()
    {
        if (_channel != null)
            _channel.OnEventRaised -= Respond;
    }

    private void Respond(T value) => _response.Invoke(value);
}

// 无参数版本
public class VoidEventListener : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO _channel;
    [SerializeField] private UnityEvent _response;

    private void OnEnable()
    {
        if (_channel != null)
            _channel.OnEventRaised += Respond;
    }

    private void OnDisable()
    {
        if (_channel != null)
            _channel.OnEventRaised -= Respond;
    }

    private void Respond() => _response.Invoke();
}

// ==================== 使用示例 ====================

// 1. 创建 Channel SO（Project窗口右键 Create/Events/Void Event Channel）
// 2. 在 Inspector 中配置引用

public class Player : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO _onPlayerDeath;
    
    public void Die()
    {
        _onPlayerDeath.RaiseEvent();
    }
}

public class GameManager : MonoBehaviour
{
    [SerializeField] private VoidEventChannelSO _onPlayerDeath;
    
    private void OnEnable() => _onPlayerDeath.OnEventRaised += HandleDeath;
    private void OnDisable() => _onPlayerDeath.OnEventRaised -= HandleDeath;
    
    private void HandleDeath() => Debug.Log("Player died!");
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| SO 作为事件中心 | ✅保留 | 核心机制，实现解耦 |
| 发布-订阅模式 | ✅保留 | 核心不变量 |
| MonoBehaviour 桥接 | ✅保留 | Inspector 配置需要 |
| 泛型 Channel | ⚠️ 简化 | 原项目为每种类型单独定义 |
| 多种业务 Channel | ❌砍掉 | 可用泛型替代 |
| **最容易搞错** | ⚠️ | 忘记在 OnDisable 中取消订阅，导致内存泄漏 |
