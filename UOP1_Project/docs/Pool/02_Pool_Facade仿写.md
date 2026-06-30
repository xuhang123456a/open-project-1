# Pool Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `IPool<T>` 接口 | 保留 | ✅保留 |
| `PoolSO<T>` 抽象基类 | 保留核心逻辑 | ✅保留 |
| `ComponentPoolSO<T>` | 保留核心逻辑 | ✅保留 |
| `Stack<T>` 存储 | 保留 | ✅保留 |
| `HasBeenPrewarmed` 标志 | 保留 | ✅保留 |
| `OnDisable` 清理 | 保留 | ✅保留 |
| Editor 销毁细节 | 砍掉 | ❌砍掉 |

---

## 最小复刻代码（~130行）

```csharp
using System.Collections.Generic;
using UnityEngine;

// ==================== 池契约 ====================

public interface IPool<T>
{
    void Prewarm(int num);
    T Request();
    void Return(T member);
}

// ==================== 通用对象池 ====================

public abstract class PoolSO<T> : ScriptableObject, IPool<T>
{
    protected readonly Stack<T> Available = new Stack<T>();
    public abstract IFactory<T> Factory { get; set; }
    protected bool HasBeenPrewarmed { get; set; }

    protected virtual T Create() => Factory.Create();

    public virtual void Prewarm(int num)
    {
        if (HasBeenPrewarmed)
        {
            Debug.LogWarning($"Pool {name} has already been prewarmed.");
            return;
        }
        for (int i = 0; i < num; i++)
            Available.Push(Create());
        HasBeenPrewarmed = true;
    }

    public virtual T Request()
    {
        return Available.Count > 0 ? Available.Pop() : Create();
    }

    public virtual void Return(T member)
    {
        Available.Push(member);
    }

    public virtual void OnDisable()
    {
        Available.Clear();
        HasBeenPrewarmed = false;
    }
}

// ==================== Component 专用池 ====================

public abstract class ComponentPoolSO<T> : PoolSO<T> where T : Component
{
    private Transform _poolRoot;
    private Transform _parent;

    private Transform PoolRoot
    {
        get
        {
            if (_poolRoot == null)
            {
                _poolRoot = new GameObject(name).transform;
                _poolRoot.SetParent(_parent);
            }
            return _poolRoot;
        }
    }

    public void SetParent(Transform t)
    {
        _parent = t;
        if (_poolRoot != null)
            PoolRoot.SetParent(_parent);
    }

    public override T Request()
    {
        T member = base.Request();
        member.gameObject.SetActive(true);
        return member;
    }

    public override void Return(T member)
    {
        member.transform.SetParent(PoolRoot);
        member.gameObject.SetActive(false);
        base.Return(member);
    }

    protected override T Create()
    {
        T newMember = base.Create();
        newMember.transform.SetParent(PoolRoot);
        newMember.gameObject.SetActive(false);
        return newMember;
    }

    public override void OnDisable()
    {
        base.OnDisable();
        if (_poolRoot != null)
            Destroy(_poolRoot.gameObject);
    }
}

// ==================== 使用示例 ====================

public class BulletPool : ComponentPoolSO<Bullet>
{
    [SerializeField] private BulletFactorySO _factory;
    public override IFactory<Bullet> Factory => _factory;
}

public class Bullet : Component
{
    public void Fire(Vector3 direction) { /* ... */ }
}

// 使用
public class Weapon : MonoBehaviour
{
    [SerializeField] private BulletPool _bulletPool;
    
    private void Start() => _bulletPool.Prewarm(20);
    
    public void Shoot()
    {
        var bullet = _bulletPool.Request();
        bullet.Fire(transform.forward);
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| Stack 复用 | ✅保留 | 核心机制 |
| Factory 注入 | ✅保留 | 核心机制 |
| Prewarm 一次性 | ✅保留 | 防止重复预热 |
| OnDisable 清理 | ✅保留 | 防止内存泄漏 |
| Component 父子关系 | ✅保留 | 场景管理需要 |
| Editor 销毁 | ❌砍掉 | 平台细节 |
| **最容易搞错** | ⚠️ | 忘记在 Return 时重置对象状态 |
