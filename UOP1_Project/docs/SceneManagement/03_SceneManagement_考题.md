# SceneManagement 考题

## 🟢 概念题

**Q1: `SceneLoader` 的 `_isLoading` 标志有什么作用？**

<details>
<summary>参考答案</summary>

防止重复加载。如果玩家同时触发两个出口，只有一个会执行。
</details>

---

**Q2: `PathStorageSO` 是如何传递路径信息的？**

<details>
<summary>参考答案</summary>

`LocationExit` 在触发时设置 `lastPathTaken`，`SpawnSystem` 在生成时读取。
</details>

---

**Q3: `InitializationLoader` 的作用是什么？**

<details>
<summary>参考答案</summary>

启动游戏：加载管理器场景，加载主菜单，卸载初始化场景。
</details>

---

**Q4: `LocationEntrance` 如何控制相机切换？**

<details>
<summary>参考答案</summary>

如果是目标入口，设置入口相机高优先级，延迟后降低优先级。
</details>

---

**Q5: `GameSceneSO` 的 `sceneType` 有什么作用？**

<details>
<summary>参考答案</summary>

区分场景类型（Location/Menu/PersistentManagers），决定加载策略。
</details>

---

## � 机制题

**Q6: 如果 `_isLoading` 为 true 时触发加载，会发生什么？**

<details>
<summary>参考答案</summary>

加载请求被忽略。
</details>

---

**Q7: `FallCatcher` 是如何工作的？**

<details>
<summary>参考答案</summary>

检测玩家掉落，设置路径，杀死玩家（触发重生）。
</details>

---

**Q8: `StartGame` 的 `ContinuePreviousGame()` 做了什么？**

<details>
<summary>参考答案</summary>

加载存档数据场景。
</details>

---

**Q9: 如果 `PathStorageSO.lastPathTaken` 为 null，`SpawnSystem` 会使用哪个出生点？**

<details>
<summary>参考答案</summary>

默认出生点。
**Q10: `SceneLoader` 为什么要禁用输入？**

<details>
<summary>参考答案</summary>

防止加载过程中玩家操作。
</details>

---

**Q11: `LocationExit` 为什么要设置 `PathStorageSO`？**

<details>
<summary>参考答案</summary>

记录玩家选择的路径，用于下次生成时选择正确的入口。
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`SceneLoader` 和 `LocationExit` 的职责分别是什么？**

<details>
<summary>参考答案</summary>

- `LocationExit`：检测触发，设置路径，发出加载请求
- `SceneLoader`：接收请求，执行加载/卸载
</details>

---

**Q13: 【漏步后果】如果忘记设置 `_isLoading = false`，会有什么问题？**

<details>
<summary>参考答案</summary>

后续加载请求被忽略。
</details>

---

**Q14: 【架构陷阱】如果两个 `LocationExit` 同时触发，会有什么问题？**

<details>
<summary>参考答案</summary>

路径可能被覆盖，加载可能重复。
</details>

---

**Q15: 【同底座差异】`PathStorageSO` 和 `TransformAnchor` 都使用 SO 存储数据，有什么区别？**

<details>
<summary>参考答案</summary>

- `PathStorageSO`：存储路径数据（值类型）
- `TransformAnchor`：存储 Transform 引用（引用类型）
</details>

---

**Q16: 【漏步后果】如果 `LocationEntrance` 忘记检查 `lastPathTaken`，会有什么问题？**

<details>
<summary>参考答案</summary>

所有入口都会尝试切换相机。
</details>

---

**Q17: 【架构陷阱】为什么使用 Addressables 而非直接 `SceneManager.LoadScene`？**

<details>
<summary>参考答案</summary>

支持异步加载、资源分包、热更新。
</details>

---

**Q18: 【漏步后果】如果 `InitializationLoader` 忘记卸载初始化场景，会有什么问题？**

<details>
<summary>参考答案</summary>

初始化场景会一直占用内存。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `LoadingScreen`，在加载时显示进度。**

<details>
<summary>参考答案</summary>

```csharp
public class LoadingScreen : MonoBehaviour
{
    [SerializeField] private Slider _progressBar;
    
    public void Show(float progress)
    {
        _progressBar.value = progress;
        gameObject.SetActive(true);
    }
    
    public void Hide() => gameObject.SetActive(false);
}
```
</details>

---

**Q20: 实现一个 `SceneTransition`，支持淡入淡出效果。**

<details>
<summary>参考答案</summary>

```csharp
public class SceneTransition : MonoBehaviour
{
    [SerializeField] private Image _fadeImage;
    [SerializeField] private float _fadeDuration = 0.5f;
    
    public IEnumerator FadeOut()
    {
        float timer = 0;
        while (timer < _fadeDuration)
        {
            timer += Time.deltaTime;
            _fadeImage.color = new Color(0, 0, 0, timer / _fadeDuration);
            yield return null;
        }
    }
    
    public IEnumerator FadeIn()
    {
        float timer = _fadeDuration;
        while (timer > 0)
        {
            timer -= Time.deltaTime;
            _fadeImage.color = new Color(0, 0, 0, timer / _fadeDuration);
            yield return null;
        }
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：切换场景后玩家位置不正确。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. `PathStorageSO` 未正确设置
2. `SpawnSystem` 未找到匹配的入口
3. 入口位置配置错误
</details>
