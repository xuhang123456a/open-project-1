# Characters Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `Protagonist` 输入消费 | 保留核心逻辑 | ✅保留 |
| `Damageable` 受伤系统 | 保留核心逻辑 | ✅保留 |
| `HealthSO` 生命值数据 | 保留 | ✅保留 |
| `Attack` 攻击碰撞体 | 保留核心逻辑 | ✅保留 |
| `Critter` 区域检测 | 保留核心逻辑 | ✅保留 |
| `ZoneTriggerController` | 保留核心逻辑 | ✅保留 |
| `DropGroup` / `DropItem` | 保留 | ✅保留 |
| StateMachine Actions/Conditions | 简化为接口 | ⚠️ 简化 |

---

## 最小复刻代码（~180行）

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

// ==================== 生命值数据 ====================

public class HealthSO : ScriptableObject
{
    [SerializeField] private int _maxHealth = 100;
    [SerializeField] private int _currentHealth = 100;
    
    public int MaxHealth => _maxHealth;
    public int CurrentHealth => _currentHealth;
    
    public void SetMaxHealth(int value) => _maxHealth = value;
    public void SetCurrentHealth(int value) => _currentHealth = value;
    public void InflictDamage(int damage) => _currentHealth -= damage;
    public void RestoreHealth(int amount)
    {
        _currentHealth += amount;
        if (_currentHealth > _maxHealth) _currentHealth = _maxHealth;
    }
}

// ==================== 可受伤组件 ====================

public class Damageable : MonoBehaviour
{
    [SerializeField] private HealthSO _healthSO;
    
    public bool IsDead { get; set; }
    public bool GetHit { get; set; }
    
    public event Action OnDie;
    
    private void Awake()
    {
        if (_healthSO == null)
        {
            _healthSO = ScriptableObject.CreateInstance<HealthSO>();
            _healthSO.SetMaxHealth(100);
            _healthSO.SetCurrentHealth(100);
        }
    }
    
    public void ReceiveAnAttack(int damage)
    {
        if (IsDead) return;
        
        _healthSO.InflictDamage(damage);
        GetHit = true;
        
        if (_healthSO.CurrentHealth <= 0)
        {
            IsDead = true;
            OnDie?.Invoke();
        }
    }
    
    public void Kill() => ReceiveAnAttack(_healthSO.CurrentHealth);
    public void Revive()
    {
        _healthSO.SetCurrentHealth(_healthSO.MaxHealth);
        IsDead = false;
    }
}

// ==================== 攻击碰撞体 ====================

public class Attack : MonoBehaviour
{
    [SerializeField] private int _attackStrength = 10;
    
    private void Awake() => gameObject.SetActive(false);
    
    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag(gameObject.tag)) return; // 友军伤害
        
        if (other.TryGetComponent(out Damageable damageable) && !damageable.GetHit)
        {
            damageable.ReceiveAnAttack(_attackStrength);
        }
    }
}

// ==================== 攻击控制器 ====================

public class Attacker : MonoBehaviour
{
    [SerializeField] private GameObject _attackCollider;
    
    public void EnableWeapon() => _attackCollider.SetActive(true);
    public void DisableWeapon() => _attackCollider.SetActive(false);
}

// ==================== 区域触发器 ====================

[Serializable]
public class BoolEvent : UnityEvent<bool, GameObject> { }

public class ZoneTriggerController : MonoBehaviour
{
    [SerializeField] private BoolEvent _enterZone;
    [SerializeField] private LayerMask _layers;
    
    private void OnTriggerEnter(Collider other)
    {
        if ((1 << other.gameObject.layer & _layers) != 0)
            _enterZone.Invoke(true, other.gameObject);
    }
    
    private void OnTriggerExit(Collider other)
    {
        if ((1 << other.gameObject.layer & _layers) != 0)
            _enterZone.Invoke(false, other.gameObject);
    }
}

// ==================== 可战斗生物 ====================

public class Critter : MonoBehaviour
{
    public bool isPlayerInAlertZone;
    public bool isPlayerInAttackZone;
    public Damageable currentTarget;
    
    public void OnAlertTriggerChange(bool entered, GameObject who)
    {
        isPlayerInAlertZone = entered;
        
        if (entered && who.TryGetComponent(out Damageable d))
        {
            currentTarget = d;
            currentTarget.OnDie += OnTargetDead;
        }
        else
        {
            currentTarget = null;
        }
    }
    
    public void OnAttackTriggerChange(bool entered, GameObject who)
    {
        isPlayerInAttackZone = entered;
    }
    
    private void OnTargetDead()
    {
        currentTarget = null;
        isPlayerInAlertZone = false;
        isPlayerInAttackZone = false;
    }
}

// ==================== 掉落系统 ====================

[Serializable]
public class DropItem
{
    public ItemSO item;
    public float dropRate;
}

[Serializable]
public class DropGroup
{
    public List<DropItem> items;
    public float groupDropRate;
}

// ==================== 使用示例 ====================

public class Enemy : MonoBehaviour
{
    [SerializeField] private Damageable _damageable;
    [SerializeField] private Attacker _attacker;
    [SerializeField] private DropGroup[] _dropGroups;
    
    private void OnEnable() => _damageable.OnDie += OnDeath;
    private void OnDisable() => _damageable.OnDie -= OnDeath;
    
    private void OnDeath()
    {
        DropRewards();
        Destroy(gameObject);
    }
    
    private void DropRewards()
    {
        foreach (var group in _dropGroups)
        {
            if (UnityEngine.Random.value < group.groupDropRate)
            {
                float cumulative = 0;
                float roll = UnityEngine.Random.value;
                foreach (var item in group.items)
                {
                    cumulative += item.dropRate;
                    if (roll < cumulative)
                    {
                        Instantiate(item.item.prefab, transform.position, Quaternion.identity);
                        break;
                    }
                }
            }
        }
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| HealthSO 实例化 | ✅保留 | 核心机制 |
| 友军伤害检查 | ✅保留 | 核心机制 |
| 掉落概率计算 | ✅保留 | 核心机制 |
| 区域触发器 | ✅保留 | 核心机制 |
| 输入消费 | ⚠️ 简化 | 只保留核心逻辑 |
| **最容易搞错** | ⚠️ | 忘记在 OnDisable 中取消订阅事件 |
