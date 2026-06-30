# Camera 考题

## 🟢 概念题

**Q1: `CameraManager` 为什么使用 `TransformAnchor` 获取玩家引用？**

<details>
<summary>参考答案</summary>

解耦。相机管理器不需要知道玩家如何生成，只需监听锚点事件。
</details>

---

**Q2: `OnEnableMouseControlCamera()` 中为什么要设置 `_cameraMovementLock`？**

<details>
<summary>参考答案</summary>

防止右键按下瞬间的鼠标移动导致相机抖动。
</details>

---

**Q3: 鼠标和手柄的输入处理有什么区别？**

<details>
<summary>参考答案</summary>

鼠标：不乘以 `Time.deltaTime`，使用固定倍数 `0.02f`
手柄：乘以 `Time.deltaTime`
</details>

---

**Q4: `OnDisableMouseControlCamera()` 中为什么要重置输入轴为 0？**

<details>
<summary>参考答案</summary>

防止松开右键后相机继续旋转（输入值"粘滞"）。
</details>

---

**Q5: `_camShakeEvent` 是如何触发相机震动的？**

<details>
<summary>参考答案</summary>

调用 `CinemachineImpulseSource.GenerateImpulse()`。
</details>

---

## 🟡 机制题

**Q6: 如果玩家生成前 `CameraManager` 已初始化，如何设置跟随目标？**

<details>
<summary>参考答案</summary>

监听 `_protagonistTransformAnchor.OnAnchorProvided`，玩家生成时自动调用 `SetupProtagonistVirtualCamera()`。
</details>

---

**Q7: 如果忘记在 `OnDisable` 中取消订阅 `OnAnchorProvided`，会有什么问题？**

<details>
<summary>参考答案</summary>

内存泄漏，已销毁对象仍响应锚点事件。
</details>

---

**Q8: `deviceMultiplier` 的计算逻辑是什么？**

<details>
<summary>参考答案</summary>

鼠标：`0.02f`（固定帧时间近似值）
手柄：`Time.deltaTime`（实际帧时间）
</details>

---

**Q9: 如果 `_cameraMovementLock` 为 true 时移动鼠标，会发生什么？**

<details>
<summary>参考答案</summary>

忽略输入，相机不旋转。
</details>

---

**Q10: `CinemachineFreeLook` 的 `Follow` 和 `LookAt` 有什么区别？**

<details>
<summary>参考答案</summary>

- `Follow`：相机跟随的目标位置
- `LookAt`：相机注视的目标位置
</details>

---

**Q11: 如果玩家在对话时移动鼠标，相机会旋转吗？**

<details>
<summary>参考答案</summary>

不会。对话时输入模式切换为 Dialogue，不启用鼠标控制相机。
</details>

---

## 🔴 架构陷阱题

**Q12: 【同底座差异】`CameraManager` 和 `TransformAnchor` 都使用 SO 存储引用，有什么区别？**

<details>
<summary>参考答案</summary>

- `TransformAnchor`：通用锚点，存储任意 Transform
- `CameraManager`：业务逻辑，控制 Cinemachine
</details>

---

**Q13: 【漏步后果】如果 `OnDisableMouseControlCamera()` 忘记重置输入轴，会有什么问题？**

<details>
<summary>参考答案</summary>

松开右键后相机继续旋转。
</details>

---

**Q14: 【架构陷阱】为什么不在 `Update` 中直接读取鼠标输入？**

<details>
<summary>参考答案</summary>

解耦。输入通过 `InputReader` 统一管理，支持输入模式切换和重绑定。
</details>

---

**Q15: 【同底座差异】`CameraManager` 和 `UIManager` 都监听事件，有什么区别？**

<details>
<summary>参考答案</summary>

- `CameraManager`：监听输入和锚点事件
- `UIManager`：监听对话、交互、暂停等业务事件
</details>

---

**Q16: 【漏步后果】如果 `SetupProtagonistVirtualCamera()` 中忘记设置 `LookAt`，会有什么问题？**

<details>
<summary>参考答案</summary>

相机可能不注视玩家，导致视角偏移。
</details>

---

**Q17: 【架构陷阱】相机震动效果是如何与游戏逻辑解耦的？**

<details>
<summary>参考答案</summary>

通过 `VoidEventChannelSO` 触发，任何系统都可以触发震动，无需直接引用 CameraManager。
</details>

---

**Q18: 【漏步后果】如果 `_speedMultiplier` 设置过大，会有什么问题？**

<details>
<summary>参考答案</summary>

相机旋转过快，玩家难以控制。
</details>

---

## ✍️ 实操题

**Q19: 实现一个 `CameraShake`，支持不同强度的震动。**

<details>
<summary>参考答案</summary>

```csharp
public class CameraShake : MonoBehaviour
{
    [SerializeField] private CinemachineImpulseSource _impulseSource;
    
    public void Shake(float intensity)
    {
        _impulseSource.GenerateImpulse(intensity);
    }
}
```
</details>

---

**Q20: 实现一个 `CameraFollow`，不使用 Cinemachine。**

<details>
<summary>参考答案</summary>

```csharp
public class CameraFollow : MonoBehaviour
{
    [SerializeField] private Transform _target;
    [SerializeField] private Vector3 _offset;
    [SerializeField] private float _smoothSpeed = 5f;
    
    private void LateUpdate()
    {
        if (_target == null) return;
        
        Vector3 desiredPosition = _target.position + _offset;
        transform.position = Vector3.Lerp(transform.position, desiredPosition, _smoothSpeed * Time.deltaTime);
        transform.LookAt(_target);
    }
}
```
</details>

---

**Q21: 假设有一个 Bug：相机不跟随玩家。请列出至少 3 个可能的原因。**

<details>
<summary>参考答案</summary>

1. `TransformAnchor` 未正确设置玩家引用
2. `SetupProtagonistVirtualCamera()` 未被调用
3. `CinemachineFreeLook` 的 `Follow`/`LookAt` 未设置
</details>