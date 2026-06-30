# Camera Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `CameraManager` 相机管理器 | 保留核心逻辑 | ✅保留 |
| `CinemachineFreeLook` | 简化为接口 | ⚠️ 简化 |
| `TransformAnchor` 锚点 | 保留 | ✅保留 |
| 鼠标控制 | 保留核心逻辑 | ✅保留 |
| 震动效果 | 保留 | ✅保留 |

---

## 最小复刻代码（~100行）

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.InputSystem;

// ==================== 相机管理器 ====================

public class CameraManager : MonoBehaviour
{
    [SerializeField] private InputReader _inputReader;
    [SerializeField] private Camera _mainCamera;
    [SerializeField] private CinemachineFreeLook _freeLookVCam;
    [SerializeField] private CinemachineImpulseSource _impulseSource;
    [SerializeField] private TransformAnchor _cameraTransformAnchor;
    [SerializeField] private TransformAnchor _protagonistTransformAnchor;
    [SerializeField] private VoidEventChannelSO _camShakeEvent;
    
    [SerializeField] private float _speedMultiplier = 1f;
    
    private bool _isRMBPressed;
    private bool _cameraMovementLock;

    private void OnEnable()
    {
        _inputReader.CameraMoveEvent += OnCameraMove;
        _inputReader.EnableMouseControlCameraEvent += OnEnableMouseControlCamera;
        _inputReader.DisableMouseControlCameraEvent += OnDisableMouseControlCamera;
        
        _protagonistTransformAnchor.OnAnchorProvided += SetupProtagonistVirtualCamera;
        _camShakeEvent.OnEventRaised += _impulseSource.GenerateImpulse;
        
        _cameraTransformAnchor.Provide(_mainCamera.transform);
    }

    private void OnDisable()
    {
        _inputReader.CameraMoveEvent -= OnCameraMove;
        _inputReader.EnableMouseControlCameraEvent -= OnEnableMouseControlCamera;
        _inputReader.DisableMouseControlCameraEvent -= OnDisableMouseControlCamera;
        
        _protagonistTransformAnchor.OnAnchorProvided -= SetupProtagonistVirtualCamera;
        _camShakeEvent.OnEventRaised -= _impulseSource.GenerateImpulse;
        
        _cameraTransformAnchor.Unset();
    }

    private void Start()
    {
        if (_protagonistTransformAnchor.isSet)
            SetupProtagonistVirtualCamera();
    }

    private void OnEnableMouseControlCamera()
    {
        _isRMBPressed = true;
        Cursor.lockState = CursorLockMode.Locked;
        Cursor.visible = false;
        StartCoroutine(DisableMouseControlForFrame());
    }

    private IEnumerator DisableMouseControlForFrame()
    {
        _cameraMovementLock = true;
        yield return new WaitForEndOfFrame();
        _cameraMovementLock = false;
    }

    private void OnDisableMouseControlCamera()
    {
        _isRMBPressed = false;
        Cursor.lockState = CursorLockMode.None;
        Cursor.visible = true;
        
        _freeLookVCam.m_XAxis.m_InputAxisValue = 0;
        _freeLookVCam.m_YAxis.m_InputAxisValue = 0;
    }

    private void OnCameraMove(Vector2 cameraMovement, bool isDeviceMouse)
    {
        if (_cameraMovementLock) return;
        if (isDeviceMouse && !_isRMBPressed) return;
        
        float deviceMultiplier = isDeviceMouse ? 0.02f : Time.deltaTime;
        
        _freeLookVCam.m_XAxis.m_InputAxisValue = cameraMovement.x * deviceMultiplier * _speedMultiplier;
        _freeLookVCam.m_YAxis.m_InputAxisValue = cameraMovement.y * deviceMultiplier * _speedMultiplier;
    }

    private void SetupProtagonistVirtualCamera()
    {
        _freeLookVCam.Follow = _protagonistTransformAnchor.Value;
        _freeLookVCam.LookAt = _protagonistTransformAnchor.Value;
    }
}

// ==================== 使用示例 ====================

public class CameraSetup : MonoBehaviour
{
    [SerializeField] private CameraManager _cameraManager;
    [SerializeField] private TransformAnchor _playerAnchor;
    
    private void Start()
    {
        // 玩家生成时
        _playerAnchor.Provide(FindObjectOfType<Player>().transform);
    }
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| Cinemachine集成 | ✅保留 | 核心机制 |
| 锚点系统 | ✅保留 | 核心机制 |
| 鼠标控制 | ✅保留 | 核心机制 |
| 震动效果 | ✅保留 | 核心机制 |
| 输入设备差异 | ✅保留 | 核心机制 |
| **最容易搞错** | ⚠️ | 忘记在OnDisable中取消订阅锚点事件 |