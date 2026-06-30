# Quests Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `QuestManagerSO` 任务管理器 | 保留核心逻辑 | ✅保留 |
| `StepController` 步骤控制器 | 保留核心逻辑 | ✅保留 |
| `QuestlineSO` / `QuestSO` / `StepSO` | 保留 | ✅保留 |
| 对话集成 | 简化 | ⚠️ 简化 |
| 事件广播 | 保留 | ✅保留 |

---

## 最小复刻代码（~150行）

```csharp
using System.Collections.Generic;
using UnityEngine;

// ==================== 任务数据 ====================

[CreateAssetMenu(menuName = "Quests/Questline")]
public class QuestlineSO : ScriptableObject
{
    public List<QuestSO> quests;
    public bool isDone;
    public System.Action OnCompleted;
    
    public void Complete()
    {
        isDone = true;
        OnCompleted?.Invoke();
    }
}

[CreateAssetMenu(menuName = "Quests/Quest")]
public class QuestSO : ScriptableObject
{
    public List<StepSO> steps;
    public bool isDone;
    public System.Action OnCompleted;
    
    public void Complete()
    {
        isDone = true;
        OnCompleted?.Invoke();
    }
}

public enum StepType
{
    Dialogue,
    GiveItem,
    CheckItem
}

[CreateAssetMenu(menuName = "Quests/Step")]
public class StepSO : ScriptableObject
{
    public string actorId;
    public StepType type;
    public ItemSO requiredItem;
    public ItemSO rewardItem;
    public bool isDone;
    
    public void Complete() => isDone = true;
}

// ==================== 任务管理器 ====================

public class QuestManager : MonoBehaviour
{
    [SerializeField] private List<QuestlineSO> _questlines;
    [SerializeField] private InventorySO _inventory;
    
    private QuestlineSO _currentQuestline;
    private QuestSO _currentQuest;
    private StepSO _currentStep;
    
    private void Start() => StartQuestline();
    
    private void StartQuestline()
    {
        foreach (var ql in _questlines)
        {
            if (!ql.isDone)
            {
                _currentQuestline = ql;
                StartQuest();
                break;
            }
        }
    }
    
    private void StartQuest()
    {
        foreach (var q in _currentQuestline.quests)
        {
            if (!q.isDone)
            {
                _currentQuest = q;
                StartStep();
                break;
            }
        }
    }
    
    private void StartStep()
    {
        foreach (var s in _currentQuest.steps)
        {
            if (!s.isDone)
            {
                _currentStep = s;
                break;
            }
        }
    }
    
    public void CheckStepValidity()
    {
        if (_currentStep == null) return;
        
        bool isValid = false;
        switch (_currentStep.type)
        {
            case StepType.Dialogue:
                isValid = true;
                break;
            case StepType.CheckItem:
                isValid = _inventory.Contains(_currentStep.requiredItem);
                break;
            case StepType.GiveItem:
                isValid = true;
                break;
        }
        
        if (isValid)
        {
            CompleteStep();
        }
    }
    
    private void CompleteStep()
    {
        _currentStep.Complete();
        
        // 给予奖励
        if (_currentStep.rewardItem != null)
        {
            _inventory.Add(_currentStep.rewardItem);
        }
        
        // 检查任务是否完成
        bool questDone = true;
        foreach (var s in _currentQuest.steps)
        {
            if (!s.isDone) { questDone = false; break; }
        }
        
        if (questDone)
        {
            CompleteQuest();
        }
        else
        {
            StartStep();
        }
    }
    
    private void CompleteQuest()
    {
        _currentQuest.Complete();
        
        // 检查任务线是否完成
        bool questlineDone = true;
        foreach (var q in _currentQuestline.quests)
        {
            if (!q.isDone) { questlineDone = false; break; }
        }
        
        if (questlineDone)
        {
            _currentQuestline.Complete();
            StartQuestline();
        }
        else
        {
            StartQuest();
        }
    }
}

// ==================== 步骤控制器 ====================

public class StepController : MonoBehaviour
{
    [SerializeField] private string _actorId;
    [SerializeField] private QuestManager _questManager;
    
    public void Interact()
    {
        _questManager.CheckStepValidity();
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 层级结构 | ✅保留 | 核心机制 |
| 状态追踪 | ✅保留 | 核心机制 |
| 物品检查 | ✅保留 | 核心机制 |
| 事件广播 | ✅保留 | 核心机制 |
| 对话集成 | ❌砍掉 | 外部系统 |
| **最容易搞错** | ⚠️ | 忘记检查任务/任务线是否完成 |
