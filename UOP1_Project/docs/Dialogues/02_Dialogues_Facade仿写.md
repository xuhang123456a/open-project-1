# Dialogues Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `DialogueManager` 对话管理器 | 保留核心逻辑 | ✅保留 |
| `DialogueDataSO` 对话数据 | 保留 | ✅保留 |
| `ActorSO` 角色定义 | `Choice` 选项 | 保留 |
| 本地化 | 简化 | ⚠️ 输入模式切换简化 |

---

## 最小复刻代码```csharp
using System.Collections.Generic;
using UnityEngine;

// ==================== 对话数据 ====================

public enum DialogueType
{
    Start,
    Completion,
    Incompletion,
    Default
}

public enum ChoiceActionType
{
    DoNothing,
    ContinueWithStep,
    WinningChoice,
    LosingChoice
}

[System.Serializable]
public class Choice
{
    public string text;
    public ChoiceActionType actionType;
    public DialogueDataSO nextDialogue;
}

[System.Serializable]
public class DialogueLine
{
    public string actorId;
    public List<string> texts;
    public List<Choice> choices;
}

[CreateAssetMenu(menuName = "Dialogues/Dialogue")]
public class DialogueDataSO : ScriptableObject
{
    public List<DialogueLine> lines;
    public DialogueType dialogueType;
}

// ==================== 对话管理器 ====================

public class DialogueManager : MonoBehaviour
{
    [SerializeField] private List<ActorSO> _actors;
    [SerializeField] private GameStateSO _gameState;
    
    public System.Action<string, ActorSO> OnLineDisplayed;
    public System.Action<List<Choice>> OnChoicesDisplayed;
    public System.Action<DialogueType> OnDialogueEnded;
    
    private DialogueDataSO _currentDialogue;
    private int _currentLineIndex;
    private int _currentTextIndex;
    
    public void StartDialogue(DialogueDataSO dialogue)
    {
        _currentDialogue = dialogue;
        _currentLineIndex = 0;
        _currentTextIndex = 0;
        
        _gameState.UpdateGameState(GameState.Dialogue);
        DisplayCurrentLine();
    }
    
    public void Advance()
    {
        _currentTextIndex++;
        
        var currentLine = _currentDialogue.lines[_currentLineIndex];
        
        if (_currentTextIndex < currentLine.texts.Count)
        {
            DisplayCurrentLine();
        }
        else if (currentLine.choices != null && currentLine.choices.Count > 0)
        {
            OnChoicesDisplayed?.Invoke(currentLine.choices);
        }
        else
        {
            _currentLineIndex++;
            _currentTextIndex = 0;
            
            if (_currentLineIndex < _currentDialogue.lines.Count)
            {
                DisplayCurrentLine();
            }
            else
            {
                EndDialogue();
            }
        }
    }
    
    public void MakeChoice(Choice choice)
    {
        switch (choice.actionType)
        {
            case ChoiceActionType.ContinueWithStep:
                // 通知任务系统
                break;
            case ChoiceActionType.WinningChoice:
                // 触发胜利
                break;
            case ChoiceActionType.LosingChoice:
                // 触发失败
                break;
        }
        
        if (choice.nextDialogue != null)
        {
            StartDialogue(choice.nextDialogue);
        }
        else
        {
            EndDialogue();
        }
    }
    
    private void DisplayCurrentLine()
    {
        var line = _currentDialogue.lines[_currentLineIndex];
        var actor = _actors.Find(a => a.actorId == line.actorId);
        OnLineDisplayed?.Invoke(line.texts[_currentTextIndex], actor);
    }
    
    private void EndDialogue()
    {
        OnDialogueEnded?.Invoke(_currentDialogue.dialogueType);
        _gameState.ResetToPreviousGameState();
    }
}

// ==================== 角色定义 ====================

[CreateAssetMenu(menuName = "Dialogues/Actor")]
public class ActorSO : ScriptableObject
{
    public string actorId;
    public Sprite portrait;
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 逐行显示 | ✅保留 | 核心机制 |
| 选项系统 | ✅保留 | 核心机制 |
| 状态切换 | ✅保留 | 核心机制 |
| 本地化 | ❌砍掉 | 平台细节 |
| **最容易搞错** | ⚠️ | 忘记在选项时取消事件订阅 |
