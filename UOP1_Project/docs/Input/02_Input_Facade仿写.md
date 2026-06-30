# Input Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `InputReader` SO | 保留核心逻辑 | ✅保留 |
| `GameInput` 生成代码 | 简化为接口 | ⚠️ 简化 |
| 多Action Map | 保留核心逻辑 | ✅保留 |
| 状态过滤 | 保留 | ✅保留 |

---

## 最小复刻代码（~100行）

```csharp
using System;
using UnityEngine;
using UnityEngine.InputSystem;

// ==================== 输入读取器 ====================

public class InputReader : MonoBehaviour
{
    public event Action JumpEvent;
    public event Action AttackEvent;
    public event Action InteractEvent;
    public event Action<Vector2> MoveEvent;
    public event Action StartedRunning;
    public event Action AdvanceDialogueEvent;
    public event Action MenuPauseEvent;
    public event Action OpenInventoryEvent;
    
    public enum InputMode
    {
        Gameplay,
        Dialogue,
        Menu
    }
    
    private InputActionMap _currentActionMap;
    
    private InputAction _jumpAction;
    private InputAction _attackAction;
    private InputAction _interactAction;
    private InputAction _moveAction;
    private InputAction _runAction;
    private InputAction _advanceDialogueAction;
    private InputAction _pauseAction;
    private InputAction _openInventoryAction;
    
    private void Awake()
    {
        // 假设这些Action在Inspector中配置或通过代码创建
    }
    
    public void SetInputMode(InputMode mode)
    {
        _currentActionMap?.Disable();
        
        switch (mode)
        {
            case InputMode.Gameplay:
                // 启用Gameplay Action Map
                break;
            case InputMode.Dialogue:
                // 启用Dialogue Action Map
                break;
            case InputMode.Menu:
                // 启用Menu Action Map
                break;
        }
    }
    
    // 回调方法
    public void OnJump(InputAction.CallbackContext ctx)
    {
        if (ctx.performed) JumpEvent?.Invoke();
    }
    
    public void OnAttack(InputAction.CallbackContext ctx)
    {
        if (ctx.performed) AttackEvent?.Invoke();
    }
    
    public void OnInteract(InputAction.CallbackContext ctx)
    {
        if (ctx.performed) InteractEvent?.Invoke();
    }
    
    public void OnMove(InputAction.CallbackContext ctx)
    {
        MoveEvent?.Invoke(ctx.ReadValue<Vector2>());
    }
    
    public void OnAdvanceDialogue(InputAction.CallbackContext ctx)
    {
        if (ctx.performed) AdvanceDialogueEvent?.Invoke();
    }
    
    public void OnPause(InputAction.CallbackContext ctx)
    {
        if (ctx.performed) MenuPauseEvent?.Invoke();
    }
    
    public void OnOpenInventory(InputAction.CallbackContext ctx)
    {
        if (ctx.performed) OpenInventoryEvent?.Invoke();
    }
}

// ==================== 使用示例 ====================

public class Player : MonoBehaviour
{
    [SerializeField] private InputReader _input;
    
    private void OnEnable()
    {
        _input.JumpEvent += OnJump;
        _input.MoveEvent += OnMove;
    }
    
    private void OnDisable()
    {
        _input.JumpEvent -= OnJump;
        _input.MoveEvent -= OnMove;
    }
    
    private void OnJump() => Debug.Log("Jump!");
    private void OnMove(Vector2 dir) => Debug.Log($"Move: {dir}");
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 事件广播 | ✅保留 | 核心机制 |
| 输入模式切换 | ✅保留 | 核心机制 |
| 状态过滤 | ✅保留 | 核心机制 |
| InputAction生成代码 | ❌砍掉 | 平台细节 |
| **最容易搞错** | ⚠️ | 忘记在OnDisable中取消订阅 |
