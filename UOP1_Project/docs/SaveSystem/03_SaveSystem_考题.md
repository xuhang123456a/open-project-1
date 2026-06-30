# SaveSystem 考题

## � 概念题

**Q1: `SaveSystem` 的备份机制是如何工作的？**

<details>
<summary>参考答案</summary>

保存前先将旧存档移动到备份文件，再写入新存档。如果写入失败，可以从备份恢复。
</details>

---

**Q2: 为什么物品存储Guid而非直接序列化引用？**

<details>
<summary>参考答案</summary>

JSON不能直接序列化SO引用。Guid是字符串，可以序列化，运行时通过Addressables加载。
</details>

---

**Q3: `LoadSavedInventory()` 为什么是协程？**

<details>
<summary>参考答案</summary>

因为Addressables加载是异步的，需要等待完成。
</details>

---

**Q4: `FileManager` 为什么是静态类？**

<details>
<summary>参考答案</summary>

无状态工具类，不需要实例化。
</details>

---

**Q5: `JsonUtility.FromJsonOverwrite()` 和 `JsonUtility.FromJson()` 有什么区别？**

<details>
<summary>参考答案</summary>

- `FromJsonOverwrite`：覆盖现有对象，保留引用
- `FromJson`：创建新对象
</details>

---

## 🟡 机制题

**Q6: 如果保存过程中崩溃，会发生什么？**

<details>
<summary>参考答案</summary>

新存档可能损坏，但备份文件完好，可以从备份恢复。
</details>

---

**Q7: 如果Guid对应的资产被删除会怎样？**

<details>
<summary>参考答案</summary>

Addressables加载失败，该物品不会出现在背包中。
</details>

---

**Q8: `SaveDataToDisk()` 中为什么要先清空列表？**

<details>
<summary>参考答案</summary>

避免重复添加。
</details>

---

**Q9: 如果两个系统同时调用 `SaveDataToDisk()`，会有什么问题？**

<details>
<summary>参考答案</summary>

文件可能损坏。需要锁机制。
</details>

---

**Q10: `Application.persistentDataPath` 在Windows上是什么路径？**

<details>
<summary>参考答案</summary>

`C:\Users\<user>\AppData\LocalLow\<company>\<product>\`
</details>

---

**Q11: `SerializedItemStack` 和 `ItemStack` 有什么区别？**

<details>
<summary>参考答案</summary>

- `ItemStack`：运行时结构体（ItemSO + Amount）
- `SerializedItemStack`：存档结构体（Guid + Amount）
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`SaveSystem` 和 `RuntimeAnchorBase` 都使用 SO 存储数据，有什么区别？**

<details>
<summary>参考答案</summary>

- `SaveSystem`：持久化到磁盘
- `RuntimeAnchorBase`：只在内存中
</details>

---

**Q13: 【漏步后果】如果忘记备份直接写入，会有什么问题？**

<details>
<summary>参考答案</summary>

写入失败时旧存档丢失。
</details>

---

**Q14: 【架构陷阱】如果存档文件被手动修改，会有什么问题？**

<details>
<summary>参考答案</summary>

可能导致加载失败或数据不一致。需要校验机制。
</details>

---

**Q15: 【同底座差异】`Save` 类和 `InventorySO` 都存储物品列表，有什么区别？**

<details>
<summary>参考答案</summary>

- `Save`：存档格式（Guid）
- `InventorySO`：运行时格式（ItemSO引用）
</details>

---

**Q16: 【漏步后果】如果 `LoadSavedInventory()` 中忘记清空背包，会有什么问题？**

<details>
<summary>参考答案</summary>

物品会重复添加。
</details>

---

**Q17: 【架构陷阱】为什么不在每次修改时自动保存？**

<details>
<summary>参考答案</summary>

性能考虑。频繁IO会影响游戏体验。
</details>

---

**Q18: 【漏步后果】如果 `MoveFile` 失败但继续写入，会有什么问题？**

<details>
<summary>参考答案</summary>

旧备份可能损坏，新写入也可能失败。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `AutoSave` 组件，定时自动保存。**

<details>
<summary>参考答案</summary>

```csharp
public class AutoSave : MonoBehaviour
{
    [SerializeField] private SaveSystem _saveSystem;
    [SerializeField] private float _interval = 60f;
    
    private float _timer;
    
    private void Update()
    {
        _timer += Time.deltaTime;
        if (_timer >= _interval)
        {
            _saveSystem.SaveToFile();
            _timer = 0;
        }
    }
}
```
</details>

---

**Q20: 实现一个 `SaveValidator`，校验存档完整性。**

<details>
<summary>参考答案</summary>

```csharp
public class SaveValidator
{
    public static bool Validate(Save save)
    {
        if (save == null) return false;
        if (save.itemStacks == null) return false;
        foreach (var item in save.itemStacks)
        {
            if (string.IsNullOrEmpty(item.itemGuid)) return false;
            if (item.amount <= 0) return false;
        }
        return true;
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：加载存档后背包为空。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. Guid不正确或资产被移动
2. Addressables加载失败
3. 忘记调用 `LoadSavedInventory()`
</details>
