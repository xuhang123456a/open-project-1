# Factory Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `IFactory<T>` 接口 | 保留 | ✅保留 |
| `FactorySO<T>` 抽象基类 | 保留 | ✅保留 |
| SO 配置 | 保留 | ✅保留 |

---

## 最小复刻代码（~50行）

```csharp
using System;

// ==================== 工厂契约 ====================

public interface IFactory<T>
{
    T Create();
}

// ==================== 抽象基类 ====================

public abstract class FactorySO<T> : ScriptableObject, IFactory<T>
{
    public abstract T Create();
}

// ==================== 具体实现示例 ====================

[CreateAssetMenu(menuName = "Factory/GameObject Factory")]
public class GameObjectFactorySO : FactorySO<GameObject>
{
    [SerializeField] private GameObject _prefab;

    public override GameObject Create()
    {
        return Instantiate(_prefab);
    }
}

[CreateAssetMenu(menuName = "Factory/Health Factory")]
public class HealthFactorySO : FactorySO<HealthSO>
{
    public override HealthSO Create()
    {
        return ScriptableObject.CreateInstance<HealthSO>();
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 泛型接口 | ✅保留 | 核心机制 |
| SO 基类 | ✅保留 | Inspector 配置需要 |
| **最容易搞错** | ⚠️ | 工厂创建的对象需要正确初始化 |
