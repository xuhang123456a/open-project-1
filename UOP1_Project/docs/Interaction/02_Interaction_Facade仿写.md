# Interaction Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `InteractionManager` 交互管理器 | 保留核心逻辑 | ✅保留 |
| `Interaction` 交互数据 | 保留 | ✅保留 |
| LinkedList优先级 | 保留 | ✅保留 |
| 类型区分 | 保留 | ✅保留 |

---

## 最小复刻代码（~100行）

```csharp
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

// ==================== 交互类型 ====================

public enum InteractionType
{
    None,
    PickUp,
    Cook,
    Talk
}

// ==================== 交互数据 ====================

public class Interaction
{
    public InteractionType type;
    public GameObject target;
    
    public Interaction(InteractionType type, GameObject target)
    {
        this.type = type;
        this.target = target;
    }
}

// ==================== 交互管理器 ====================

public class InteractionManager : MonoBehaviour
{
    public UnityEvent<ItemSO> OnItemPickup;
    public UnityEvent OnCookingStart;
    public UnityEvent<ActorSO> OnTalkStart;
    public UnityEvent<bool> OnInteractionUIChanged;
    
    private LinkedList<Interaction> _interactions = new LinkedList<Interaction>();
    
    private void OnEnable()
    {
        // 订阅输入事件
    }
    
    private void OnDisable()
    {
        // 取消订阅
    }
    
    public void OnInteractPressed()
    {
        if (_interactions.Count == 0) return;
        
        var first = _interactions.First.Value;
        
        switch (first.type)
        {
            case InteractionType.PickUp:
                // 等待动画调用 Collect()
                break;
            case InteractionType.Cook:
                OnCookingStart?.Invoke();
                break;
            case InteractionType.Talk:
                var actor = first.target.GetComponent<ActorSO>();
                OnTalkStart?.Invoke(actor);
                break;
        }
    }
    
    public void Collect()
    {
        if (_interactions.Count == 0) return;
        
        var first = _interactions.First.Value;
        _interactions.RemoveFirst();
        
        // 拾取物品
        var collectable = first.target.GetComponent<CollectableItem>();
        OnItemPickup?.Invoke(collectable.GetItem());
        
        Destroy(first.target);
        UpdateUI();
    }
    
    public void OnTriggerChanged(bool entered, GameObject obj)
    {
        if (entered)
            AddInteraction(obj);
        else
            RemoveInteraction(obj);
    }
 AddInteraction(GameObject obj)
    {
        var type = InteractionType.None;
        
        if (obj.CompareTag("Pickable")) type = InteractionType.PickUp;
        else if (obj.CompareTag("Cookable")) type = InteractionType.Cook;
        else if (obj.CompareTag("Talkable")) type = InteractionType.Talk;
        
        _interactions.AddLast(new Interaction(type, obj));
        UpdateUI();
    }
    
    private void RemoveInteraction(GameObject obj)
    {
        var node = _interactions.First;
        while (node != null)
        {
            if (node.Value.target == obj)
            {
                _interactions.Remove(node);
                break;
            }
            node = node.Next;
        }
        UpdateUI();
    }
    
    private void UpdateUI()
    {
        bool show = _interactions.Count > 0;
        OnInteractionUIChanged?.Invoke(show);
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| LinkedList优先级 | ✅保留 | 核心机制 |
| 类型区分 | ✅保留 | 核心机制 |
| 事件驱动 | ✅保留 | 核心机制 |
| **最容易搞错** | ⚠️ | 忘记在Remove时正确遍历链表 |
