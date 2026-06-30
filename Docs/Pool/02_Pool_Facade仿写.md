# Pool 模块 Facade 仿写

> 目标：用纯 C#（不依赖 Unity）复刻池的**核心不变量**——工厂注入创建、栈式借还、预热一次、整池清理。
> 砍掉 ScriptableObject 资产化、Component 激活态、平台分支。

---

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `IFactory<T>.Create()` | `Func<T>` 委托 | ✅ 保留（用委托替接口，更轻）|
| `FactorySO<T> : ScriptableObject` | 删除，工厂用委托注入 | ❌ 砍掉（无 SO 资产概念）|
| `PoolSO<T> : ScriptableObject` | `Pool<T>` 普通类 | ⚠️ 简化：失去「资产即共享」，改由调用方持有实例 |
| `Stack<T> Available` | 同 | ✅ 保留（LIFO 复用）|
| `HasBeenPrewarmed` 守卫 | 同 | ✅ 保留（核心不变量：预热一次）|
| `Request(int)` 批量 | 保留 | ✅ 保留 |
| `ComponentPoolSO` 激活态/父节点 | `IPoolable` 接口的 `OnRequest/OnReturn` 钩子 | ⚠️ 抽象化：用钩子代替 Unity 的 SetActive |
| `OnDisable` 整池清理 | `Dispose()` | ✅ 保留语义（手动触发）|
| 无借出追踪 | **新增** `HashSet` 幂等校验（健壮性增强）| ⚠️ 原版砍掉，这里补回以演示难点 3 |

---

## 最小复刻代码（独立可编译，~150 行）

```csharp
using System;
using System.Collections.Generic;

namespace MiniPool
{
    /// <summary>可被池管理的对象可选实现此接口，获得借出/归还时机回调。</summary>
    public interface IPoolable
    {
        void OnRequest();  // 借出：等价于 Unity 的 SetActive(true)
        void OnReturn();   // 归还：等价于 SetActive(false)
    }

    public interface IPool<T>
    {
        void Prewarm(int num);
        T Request();
        void Return(T member);
    }

    /// <summary>
    /// 通用对象池：工厂委托按需创建，栈式 LIFO 复用，预热一生一次。
    /// 核心不变量：
    ///   (1) Available 中的对象都是"空闲可借"的；
    ///   (2) Prewarm 只生效一次；
    ///   (3) Return 幂等——重复归还同一对象不会污染栈。
    /// </summary>
    public class Pool<T> : IPool<T>, IDisposable
    {
        protected readonly Stack<T> Available = new Stack<T>();
        // 原版为性能砍掉了借出追踪；这里补回以防"double free"（落地难点3）
        private readonly HashSet<T> _inPool = new HashSet<T>();
        private readonly Func<T> _factory;
        protected bool HasBeenPrewarmed { get; private set; }

        public Pool(Func<T> factory)
        {
            _factory = factory ?? throw new ArgumentNullException(nameof(factory));
        }

        protected virtual T Create() => _factory();

        public virtual void Prewarm(int num)
        {
            if (HasBeenPrewarmed)
            {
                Console.WriteLine("[Pool] already prewarmed, ignored.");
                return;
            }
            for (int i = 0; i < num; i++)
                PushInternal(Create());
            HasBeenPrewarmed = true;
        }

        public virtual T Request()
        {
            T member = Available.Count > 0 ? PopInternal() : Create();
            (member as IPoolable)?.OnRequest();
            return member;
        }

        public IEnumerable<T> Request(int num)
        {
            var list = new List<T>(num);
            for (int i = 0; i < num; i++) list.Add(Request());
            return list;
        }

        public virtual void Return(T member)
        {
            if (member == null) return;
            if (_inPool.Contains(member)) return;       // 幂等：已在池中，忽略重复归还
            (member as IPoolable)?.OnReturn();
            PushInternal(member);
        }

        public void Return(IEnumerable<T> members)
        {
            foreach (var m in members) Return(m);
        }

        private void PushInternal(T m) { Available.Push(m); _inPool.Add(m); }
        private T PopInternal() { var m = Available.Pop(); _inPool.Remove(m); return m; }

        public virtual void Dispose()
        {
            Available.Clear();
            _inPool.Clear();
            HasBeenPrewarmed = false;
        }
    }
}
```

---

## 使用示例

```csharp
using System;
using MiniPool;

class Bullet : IPoolable
{
    public int Id;
    public void OnRequest() => Console.WriteLine($"Bullet {Id} 激活");
    public void OnReturn()  => Console.WriteLine($"Bullet {Id} 失活回池");
}

class Demo
{
    static void Main()
    {
        int seq = 0;
        var pool = new Pool<Bullet>(() => new Bullet { Id = seq++ }); // 工厂注入

        pool.Prewarm(2);                 // 预创建 Bullet 0,1（已 OnReturn 失活态进栈）
        var b = pool.Request();          // 复用 Bullet 1（LIFO），打印"激活"
        pool.Return(b);                  // 归还
        pool.Return(b);                  // 重复归还 → 被幂等忽略（原版会污染栈）

        var batch = pool.Request(3);     // 借 3 个：复用 2 个 + 新建 1 个
        pool.Return(batch);
        pool.Dispose();                  // 整池清理
    }
}
```

---

## 取舍自检

- ✅ **保留**：工厂注入（委托）、栈式 LIFO、`Prewarm` 一次性守卫、批量借还、`Dispose` 整池清理、`IPoolable` 借还钩子（对应 Component 激活态）。
- ❌ **砍掉**：ScriptableObject 资产化（失去「磁盘资产即共享实例」，改由调用方持有 `Pool` 实例）、`ComponentPoolSO` 的 Transform 父子层级、`UNITY_EDITOR` 平台分支、`OnDisable` 自动触发（改手动 `Dispose`）。
- ⚠️ **最容易搞错的一处**：原版 `Return` **不做幂等校验**（`Available.Push(member)` 直接压栈）。重复归还同一对象会让它在栈里出现两次，随后被两次 `Request` 借给两个调用方——双方都以为独占，状态互相踩踏。本仿写用 `_inPool` 这个 `HashSet` 补回幂等性；若要 1:1 复刻原版行为则删掉它，但必须在调用方严格保证「借一次还一次」。
