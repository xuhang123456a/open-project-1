# StateMachine 模块 Facade 仿写

> 目标：复刻四个核心不变量——①定义层/运行时层分离 + 一次性构建；②享元去重缓存；③单帧条件缓存与统一清除；④与/或分组求值。
> 砍掉 ScriptableObject 资产化、Unity 组件缓存、编辑器调试器、程序集重载处理。

---

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `TransitionTableSO`(ScriptableObject) | `TransitionTable` 普通类 + `TransitionDef` 列表 | ⚠️ 去 SO，定义用普通对象 |
| `createdInstances` 享元缓存 | `Dictionary<object,object>` | ✅ 保留（核心）|
| `StateSO/ActionSO/ConditionSO` 工厂 | `Func<>` 工厂 或 定义对象自带 `Create` | ✅ 保留工厂注入 |
| `Condition` 单帧缓存 + `ClearStatementCache` | 同 | ✅ 保留（核心）|
| `StateTransition` 与/或 `_resultGroups` | 同 | ✅ 保留（核心）|
| `_originSO` 享元内蕴 | `Def` 反向引用 | ✅ 保留 |
| `StateMachine.TryGetComponent` 缓存 | 删除 | ❌ 砍掉（非通用）|
| `StateMachineDebugger` | 删除 | ❌ 砍掉 |
| `Update` 每帧驱动 | `Tick()` 手动驱动 | ⚠️ 用 Tick 替 Unity Update |

---

## 最小复刻代码（独立可编译，~175 行）

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace MiniSM
{
    public enum Op { And, Or }

    // ---------- 运行时层 ----------
    public abstract class Condition
    {
        private bool _cached; private bool _value;
        public abstract bool Statement();
        public bool GetStatement()           // 单帧缓存
        {
            if (!_cached) { _cached = true; _value = Statement(); }
            return _value;
        }
        public void ClearCache() => _cached = false;
        public virtual void Awake() { }
    }

    // 条件 + 期望值 的不可变快照
    public readonly struct CondUsageRT
    {
        public readonly Condition Condition;
        public readonly bool Expected;
        public CondUsageRT(Condition c, bool e) { Condition = c; Expected = e; }
        public bool IsMet() => Condition.GetStatement() == Expected;
    }

    public abstract class StateAction
    {
        public virtual void OnEnter() { }
        public virtual void OnUpdate() { }
        public virtual void OnExit() { }
        public virtual void Awake() { }
    }

    public sealed class Transition
    {
        private readonly State _target;
        private readonly CondUsageRT[] _conds;
        private readonly int[] _groups;   // 每个 OR 组含几个 AND
        private readonly bool[] _results;
        public Transition(State target, CondUsageRT[] conds, int[] groups)
        {
            _target = target; _conds = conds;
            _groups = (groups != null && groups.Length > 0) ? groups : new int[1];
            _results = new bool[_groups.Length];
        }
        public bool TryGet(out State s) { s = ShouldTransition() ? _target : null; return s != null; }
        private bool ShouldTransition()
        {
            int count = _groups.Length;
            for (int i = 0, idx = 0; i < count && idx < _conds.Length; i++)
                for (int j = 0; j < _groups[i]; j++, idx++)
                    _results[i] = j == 0 ? _conds[idx].IsMet() : _results[i] && _conds[idx].IsMet();
            bool ret = false;
            for (int i = 0; i < count && !ret; i++) ret = ret || _results[i];
            return ret;
        }
        public void ClearCache() { foreach (var c in _conds) c.Condition.ClearCache(); }
    }

    public sealed class State
    {
        public Transition[] Transitions = Array.Empty<Transition>();
        public StateAction[] Actions = Array.Empty<StateAction>();
        public string Name;
        public void OnEnter() { foreach (var a in Actions) a.OnEnter(); }
        public void OnUpdate() { foreach (var a in Actions) a.OnUpdate(); }
        public void OnExit() { foreach (var a in Actions) a.OnExit(); }
        public bool TryGetTransition(out State next)
        {
            next = null;
            foreach (var t in Transitions) if (t.TryGet(out next)) break;
            foreach (var t in Transitions) t.ClearCache();   // 统一清除（核心不变量）
            return next != null;
        }
    }

    // ---------- 定义层 ----------
    public sealed class StateDef
    {
        public string Name;
        public List<Func<StateAction>> ActionFactories = new();
    }
    public sealed class CondDef
    {
        public Func<Condition> Factory; public bool Expected; public Op Operator;
    }
    public sealed class TransitionDef
    {
        public StateDef From, To; public List<CondDef> Conditions = new();
    }

    // ---------- 构建器（享元去重 + 与或分组压缩）----------
    public sealed class TransitionTable
    {
        public List<TransitionDef> Transitions = new();
        public State BuildInitial()
        {
            var cache = new Dictionary<object, object>();   // 享元缓存
            var states = new List<State>();
            foreach (var grp in Transitions.GroupBy(t => t.From))
            {
                var state = GetState(grp.Key, cache); states.Add(state);
                var list = new List<Transition>();
                foreach (var td in grp)
                {
                    var to = GetState(td.To, cache);
                    var (conds, groups) = ProcessConditions(td.Conditions, cache);
                    list.Add(new Transition(to, conds, groups));
                }
                state.Transitions = list.ToArray();
            }
            return states.Count > 0 ? states[0] : throw new InvalidOperationException("empty table");
        }
        private State GetState(StateDef def, Dictionary<object, object> cache)
        {
            if (cache.TryGetValue(def, out var o)) return (State)o;
            var s = new State { Name = def.Name }; cache[def] = s;     // 先入缓存防环
            s.Actions = def.ActionFactories.Select(f => GetAction(f, cache)).ToArray();
            return s;
        }
        private StateAction GetAction(Func<StateAction> f, Dictionary<object, object> cache)
        {
            if (cache.TryGetValue(f, out var o)) return (StateAction)o;
            var a = f(); cache[f] = a; a.Awake(); return a;
        }
        private Condition GetCondition(Func<Condition> f, Dictionary<object, object> cache)
        {
            if (cache.TryGetValue(f, out var o)) return (Condition)o;
            var c = f(); cache[f] = c; c.Awake(); return c;
        }
        private (CondUsageRT[], int[]) ProcessConditions(List<CondDef> defs, Dictionary<object, object> cache)
        {
            int n = defs.Count;
            var usages = new CondUsageRT[n];
            for (int i = 0; i < n; i++)
                usages[i] = new CondUsageRT(GetCondition(defs[i].Factory, cache), defs[i].Expected);
            var groups = new List<int>();
            for (int i = 0; i < n; i++)
            {
                int idx = groups.Count; groups.Add(1);
                while (i < n - 1 && defs[i].Operator == Op.And) { i++; groups[idx]++; }
            }
            return (usages, groups.ToArray());
        }
    }

    // ---------- 宿主 ----------
    public sealed class StateMachine
    {
        private State _current;
        public StateMachine(TransitionTable table) => _current = table.BuildInitial();
        public void Start() => _current.OnEnter();
        public void Tick()
        {
            if (_current.TryGetTransition(out var next)) { _current.OnExit(); _current = next; _current.OnEnter(); }
            _current.OnUpdate();
        }
        public string CurrentName => _current.Name;
    }
}
```

---

## 使用示例

```csharp
using System;
using MiniSM;

class IsHungry : Condition { public static bool Hungry; public override bool Statement() => Hungry; }
class Idle : StateAction { public override void OnUpdate() => Console.WriteLine("发呆..."); }
class Eat  : StateAction { public override void OnEnter() => Console.WriteLine(">> 开吃"); }

class Demo
{
    static void Main()
    {
        var idle = new StateDef { Name = "Idle" };  idle.ActionFactories.Add(() => new Idle());
        var eat  = new StateDef { Name = "Eat" };    eat.ActionFactories.Add(() => new Eat());

        var table = new TransitionTable();
        // Idle --(IsHungry==true)--> Eat
        table.Transitions.Add(new TransitionDef { From = idle, To = eat,
            Conditions = { new CondDef { Factory = () => new IsHungry(), Expected = true, Operator = Op.Or } } });

        var sm = new StateMachine(table);
        sm.Start();
        sm.Tick();                       // Idle: 发呆
        IsHungry.Hungry = true;
        sm.Tick();                       // 转换到 Eat → 打印 ">> 开吃"，本帧跑 Eat.OnUpdate（无输出）
        Console.WriteLine(sm.CurrentName); // Eat
    }
}
```

---

## 取舍自检

- ✅ **保留**：定义层/运行时层分离 + `BuildInitial` 一次性构建、`createdInstances` 享元去重（含「先入缓存防环」）、`Condition` 单帧缓存 + `TryGetTransition` 末尾统一清除、`_resultGroups` 与/或分组压缩与求值、Tick 中「先转换后 OnUpdate（本帧跑新状态）」。
- ❌ **砍掉**：ScriptableObject 资产化、`StateMachine.TryGetComponent/GetOrAddComponent` 组件缓存、`StateMachineDebugger`、`AssemblyReloadEvents` 重载重建、`SpecificMoment` 枚举。
- ⚠️ **最容易搞错的一处**：享元构建里 `GetState` 必须**先把空 State 放进缓存再去构建它的 Actions/Transitions**（`cache[def]=s` 在填充之前）。若顺序反了，当状态图存在环（A→B→A）时会无限递归 / 重复 new。其次易错：`ClearCache` 必须在「遍历完所有 transition 之后」而非每条之后——否则跨转换共享的条件会被重复求值，破坏单帧缓存优化。
