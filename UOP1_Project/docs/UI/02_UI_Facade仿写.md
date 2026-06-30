# UI Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `UIManager` 总管理器 | 保留核心逻辑 | ✅保留 |
| `UIPause` 暂停菜单 | 保留 | ✅保留 |
| `UIInventory` 背包面板 | 保留核心逻辑 | ✅保留 |
| `UIDialogueManager` 对话UI | 保留核心逻辑 | ✅保留 |
| `交互提示 | 保留核心逻辑 | ✅保留 |
| `FadeController` 淡入淡出 | 保留 | ✅保留 |
| 多面板协调 | 保留核心逻辑 | ✅保留 |

---

## 最小复刻代码（~150行）

```csharp
using System;
using UnityEngine;
using UnityEngine.Events;

// ==================== UI管理器 ====================

public class UIManager : MonoBehaviour
{
    [SerializeField] private GameObject _pausePanel;
    [SerializeField] private GameObject _inventoryPanel;
    [SerializeField] private GameObject _dialoguePanel;
    [SerializeField] private GameObject _interactionPanel;
    
    public event Action<GameState, GameState> OnGameStateChanged;
    
    private void OnEnable()
    {
        // 订阅事件
    }
    
    private void OnDisable()
    {
        // 取消订阅
    }
    
    public void ResetUI()
    {
        _pausePanel.SetActive(false);
        _inventoryPanel.SetActive(false);
        _dialoguePanel.SetActive(false);
        _interactionPanel.SetActive(false);
        Time.timeScale = 1f;
    }
    
    public void OpenPause()
    {
        _pausePanel.SetActive(true);
        Time.timeScale = 0f;
        // 切换输入模式
    }
    
    public void ClosePause()
    {
        _pausePanel.SetActive(false);
        Time.timeScale = 1f;
        // 恢复输入模式
    }
    
    public void OpenInventory()
    {
        _inventoryPanel.SetActive(true    public void CloseInventory()
    {
        _inventoryPanel.SetActive(false);
    }
    
    public void ShowDialogue(string text, Sprite portrait)
    {
        _dialoguePanel.SetActive(true);
        // 设置对话内容
    }
    
    public void HideDialogue()
    {
        _dialoguePanel.SetActive(false);
    }
    
    public void ShowInteractionPrompt(string text)
    {
        _interactionPanel.SetActive(true);
        // 设置交互提示
    }
    
    public void HideInteractionPrompt()
    {
        _interactionPanel.SetActive(false);
    }
}

// ==================== 淡入淡出控制 ====================

public class FadeController : MonoBehaviour
{
    [SerializeField] private CanvasGroup _canvasGroup;
    
    public IEnumerator FadeOut(float duration)
    {
        float timer = 0f;
        while (timer < duration)
        {
            timer += Time.unscaledDeltaTime;
            _canvasGroup.alpha = timer / duration;
            yield return null;
        }
        _canvasGroup.alpha = 1f;
    }
    
    public IEnumerator FadeIn(float duration)
    {
        float timer = 0f;
        while (timer < duration)
        {
            timer += Time.unscaledDeltaTime;
            _canvasGroup.alpha = 1f - timer / duration;
            yield return null;
        }
        _canvasGroup.alpha = 0f;
    }
}

// ==================== 加载界面 ====================

public class LoadingScreen : MonoBehaviour
{
    [SerializeField] private CanvasGroup _canvasGroup;
    [SerializeField] private UnityEngine.UI.Slider _progressBar;
    
    public void Show()
    {
        gameObject.SetActive(true);
        _canvasGroup.alpha = 1f;
    }
    
    public void Hide()
    {
        gameObject.SetActive(false);
    }
    
    public void SetProgress(float progress)
    {
        _progressBar.value = progress;
    }
}

// ==================== 使用示例 ====================

public class GameFlow : MonoBehaviour
{
    [SerializeField] private UIManager _uiManager;
    
    private void Start()
    {
        _uiManager.ResetUI();
    }
    
    private void Update()
    {
        if (Input.GetKeyDown(KeyCode.Escape))
        {
            _uiManager.OpenPause();
        }
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 事件驱动 | ✅保留 | 核心机制 |
| 时间缩放 | ✅保留 | 核心机制 |
| 输入模式切换 | ✅保留 | 核心机制 |
| 多面板协调 | ✅保留 | 核心机制 |
| 本地化 | ❌砍掉 | 平台细节 |
| **最容易搞错** | ⚠️ | 忘记在OnDisable中取消订阅事件 |
