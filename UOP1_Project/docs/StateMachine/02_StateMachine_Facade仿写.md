# StateMachine Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `TransitionTableSO` + `StateSO` + `StateActionSO` + `StateConditionSO` 四层SO | 合并为 `StateConfig` + `ActionConfig` + `ConditionConfig` | ⚠️ 简化 |
| `Dictionary<ScriptableObject, object>` 实例缓存 | 保留，使用 `Dictionary<object, object>` | ✅保留 |
| `StateTransition` + `StateCondition` + `Condition` 三层结构 | 合并为 `Transition` + `Condition` | ⚠️ 简化 |
| `resultGroups` AND/OR 分组 | 保留核心逻辑 | ✅保留 |
| `Condition` 缓存机制 | 保留 `_isCached` + `ClearCache` | ✅保留 |
| `StateMachine` MonoBehaviour 宿主 | 保留核心 Update 驱动 | ✅保留 |
| `GetOrAddComponent` / `TryGetComponent` 缓存 | 砍掉（Unity细节） | ❌砍掉 |
| Editor 重载支持 | 砍掉 | ❌砍掉 |
| `SpecificMoment` 枚举 | 砍掉（只保留 OnUpdate） | ❌砍掉 |

---

## 最小复刻代码（~150行）

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ==================== 核心契约 ====================

public interface IStateComponent
{
    void OnStateEnter();
    void OnStateExit();
}

// ==================== 条件系统 ====================

public abstract class Condition
{
    private bool _isCached;
    private bool _cachedValue;

    protected abstract bool Evaluate();

    public bool GetResult()
    {
        if (!_isCached)
        {
            _isCached = true;
            _cachedValue = Evaluate();
        }
        return _cachedValue;
    }

    public void ClearCache() => _isCached = false;
}

// ==================== 动作系统 ====================

public abstract class StateAction : IStateComponent
{
    public abstract void OnUpdate();
    public virtual void OnStateEnter() { }
    public virtual void OnStateExit() { }
}

// ==================== 转换与状态 ====================

public class Transition : IStateComponent
{
    public State Target { get; }
    private readonly List<Condition> _conditions;
    private readonly bool _requireAll; // true=AND, false=OR

    public Transition(State Target, List<Condition> conditions, bool requireAll)
    {
        this.Target = Target;
        _conditions = conditions;
        _requireAll = requireAll;
    }

    public bool IsMet()
    {
        if (_conditions.Count == 0) return true;
        return _requireAll
            ? _conditions.All(c => c.GetResult())
            : _conditions.Any(c => c.GetResult());
    }

    public void ClearCache() => _conditions.ForEach(c => c.ClearCache());
    public void OnStateEnter() { }
    public void OnStateExit() { }
}

public class State : IStateComponent
{
    public string Name { get; }
    private readonly List<StateAction> _actions;
    private readonly List<Transition> _transitions;

    public State(string name, List<StateAction> actions, List<Transition> transitions)
    {
        Name = name;
        _actions = actions;
        _transitions = transitions;
    }

    public bool TryGetTransition(out State nextState)
    {
        nextState = null;
        foreach (var transition in _transitions)
        {
            if (transition.IsMet())
            {
                nextState = transition.Target;
                break;
            }
        }
        _transitions.ForEach(t => t.ClearCache());
        return nextState != null;
    }

    public void OnStateEnter() => _actions.ForEach(a => a.OnStateEnter());
    public void OnUpdate() => _actions.ForEach(a => a.OnUpdate());
    public void OnStateExit() => _actions.ForEach(a => a.OnStateExit());
}

// ==================== 状态机 ====================

public class StateMachine
{
    public State CurrentState { get; private set; }

    public StateMachine(State initialState) => CurrentState = initialState;

    public void Update()
    {
        if (CurrentState.TryGetTransition(out var next))
        {
            CurrentState.OnStateExit();
            CurrentState = next;
            CurrentState.OnStateEnter();
        }
        CurrentState.OnUpdate();
    }
}

// ==================== 使用示例 ====================

// 定义条件
public class HealthBelowCondition : Condition
{
    private float _threshold;
    private Func<float> _getHealth;
    public HealthBelowCondition(float threshold, Func<float> getHealth)
    {
        _threshold = threshold;
        _getHealth = getHealth;
    }
    protected override bool Evaluate() => _getHealth() < _threshold;
}

// 定义动作
public class MoveAction : StateAction
{
    public override void OnUpdate() => Console.WriteLine("Moving...");
}

public class AttackAction : StateAction
{
    public override void OnStateEnter() => Console.WriteLine("Attack start!");
}

// 构建状态机
public class Demo
{
    public static void Main()
    {
        var idle = new State("Idle", new List<StateAction> { new MoveAction() }, new List<Transition>());
        var attack = new State("Attack", new List<StateAction> { new AttackAction() }, new List<Transition>());
        
        var healthCond = new HealthBelowCondition(50, () => 30);
        idle._transitions.Add(new Transition(attack, new List<Condition> { healthCond }, requireAll: true));
        
        var sm = new StateMachine(idle);
        idle.OnStateEnter();
        sm.Update(); // 如果血量<50，会转换到Attack
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| SO配置驱动 | ❌砍掉 | 仿写聚焦运行时核心，SO序列化是Unity特有 |
| 实例1:1缓存 | ✅保留 | 核心不变量，防止重复创建 |
| 条件缓存机制 | ✅保留 | 核心不变量，避免重复计算 |
| AND/OR分组 | ✅保留 | 核心逻辑，简化为requireAll参数 |
| 组件缓存 | ❌砍掉 | Unity-specific优化 |
| Editor重载 | ❌砍掉 | Editor-only功能 |
| **最容易搞错** | ⚠️ | `ClearCache()` 必须在所有Transition检查完毕后调用，而非在单个Transition内部 |
