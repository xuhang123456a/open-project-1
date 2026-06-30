# Events 模块 Facade 仿写

> 目标：复刻「事件通道作为解耦中枢」+「Listener 适配器」+「带返回值的请求通道」+「RuntimeAnchor 通知槽」四个核心不变量。
> 用纯 C# 表达，砍掉 ScriptableObject 资产化与 Unity 生命周期回调（改为显式 Attach/Detach）。

---

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `*EventChannelSO : ScriptableObject` | `EventChannel` / `EventChannel<T>` 普通类 | ⚠️ 失去资产共享，改由调用方持有同一实例 |
| `public UnityAction OnEventRaised`（裸字段）| `event Action`（用 event 增强封装）| ⚠️ 增强：防外部抹除订阅，演示落地难点3 |
| `RaiseEvent` 判空 Invoke | `Raise()` | ✅ 保留判空 |
| `LoadEventChannelSO` 无人接听告警 | `RaiseOrWarn()` | ✅ 保留（漏接即报警）|
| `AudioCueEventChannelSO` 带返回值 | `RequestChannel<TArg,TResult>` | ✅ 保留（请求-应答语义）|
| `VoidEventListener`(OnEnable/OnDisable) | `Listener`(Attach/Detach + IDisposable) | ⚠️ 用显式方法替 Unity 回调 |
| `RuntimeAnchorBase<T>` | `RuntimeAnchor<T>` | ✅ 保留 Provide/Unset/通知 |

---

## 最小复刻代码（独立可编译，~165 行）

```csharp
using System;

namespace MiniEvents
{
    // ---------- 无参/带参事件通道（广播，无返回值）----------
    public class EventChannel
    {
        public event Action OnRaised;
        public void Raise() => OnRaised?.Invoke();
        // 对应 LoadEventChannelSO：漏接是严重错误时报警
        public void RaiseOrWarn(string ctx)
        {
            if (OnRaised != null) OnRaised.Invoke();
            else Console.WriteLine($"[Channel] '{ctx}' raised but nobody listening!");
        }
    }

    public class EventChannel<T>
    {
        public event Action<T> OnRaised;
        public void Raise(T payload) => OnRaised?.Invoke(payload);
    }

    // ---------- 带返回值的请求通道（请求-应答，对应 AudioCueEventChannelSO）----------
    // 只允许一个"应答者"，返回其结果；无应答者时给出哨兵值。
    public class RequestChannel<TArg, TResult>
    {
        public Func<TArg, TResult> OnRequested;
        private readonly TResult _invalid;
        public RequestChannel(TResult invalidValue) => _invalid = invalidValue;

        public TResult Request(TArg arg)
        {
            if (OnRequested != null) return OnRequested.Invoke(arg);
            Console.WriteLine("[RequestChannel] no responder, return Invalid.");
            return _invalid;
        }
    }

    // ---------- Listener 适配器（通道 -> 本地响应）----------
    // 对应 VoidEventListener：Attach 订阅、Detach 退订，保证对称。
    public sealed class Listener : IDisposable
    {
        private readonly EventChannel _channel;
        private readonly Action _response;
        private bool _attached;

        public Listener(EventChannel channel, Action response)
        {
            _channel = channel; _response = response;
        }
        public void Attach()   // 等价 OnEnable
        {
            if (_attached || _channel == null) return;
            _channel.OnRaised += _response; _attached = true;
        }
        public void Detach()   // 等价 OnDisable
        {
            if (!_attached || _channel == null) return;
            _channel.OnRaised -= _response; _attached = false;
        }
        public void Dispose() => Detach();  // 防漏退订
    }

    // ---------- RuntimeAnchor：带通知的全局引用槽 ----------
    public class RuntimeAnchor<T> where T : class
    {
        public event Action OnAnchorProvided;
        public bool IsSet { get; private set; }
        public T Value { get; private set; }

        public void Provide(T value)
        {
            if (value == null) { Console.WriteLine("[Anchor] null provided, ignored."); return; }
            Value = value; IsSet = true;
            OnAnchorProvided?.Invoke();
        }
        public void Unset() { Value = null; IsSet = false; }
    }
}
```

---

## 使用示例

```csharp
using System;
using MiniEvents;

class Demo
{
    static void Main()
    {
        // 1) 广播通道：发布者与订阅者只共享 channel 实例（模拟同一份 SO 资产）
        var exitGame = new EventChannel();
        var listener = new Listener(exitGame, () => Console.WriteLine("收到退出事件"));
        listener.Attach();
        exitGame.RaiseOrWarn("ExitGame");   // 打印"收到退出事件"
        listener.Detach();
        exitGame.RaiseOrWarn("ExitGame");   // 无人接听 → 报警

        // 2) 请求-应答通道：play 返回一个 key（模拟 AudioCueKey）
        var audio = new RequestChannel<string, int>(invalidValue: -1);
        audio.OnRequested = clip => { Console.WriteLine($"播放 {clip}"); return 42; };
        int key = audio.Request("explosion.wav");  // 返回 42
        Console.WriteLine($"控制句柄 key={key}");

        // 3) RuntimeAnchor：玩家运行时生成后 Provide，相机订阅
        var playerAnchor = new RuntimeAnchor<string>();
        playerAnchor.OnAnchorProvided += () => Console.WriteLine($"相机锁定玩家 {playerAnchor.Value}");
        playerAnchor.Provide("PigChef(Clone)");     // 触发通知
        Console.WriteLine($"isSet={playerAnchor.IsSet}");
    }
}
```

---

## 取舍自检

- ✅ **保留**：通道作为发布/订阅唯一共享物（解耦中枢）、`Raise` 判空、漏接告警、请求-应答带返回值通道、Listener 的 Attach/Detach 对称订阅、RuntimeAnchor 的 Provide/Unset/通知。
- ❌ **砍掉**：ScriptableObject 资产化（改为持有同一实例）、`DescriptionBaseSO`/GUID、Unity 的 `OnEnable/OnDisable` 自动时机（改显式调用 + `IDisposable`）、几十个具体类型通道（用泛型 `EventChannel<T>` 归一）。
- ⚠️ **最容易搞错的一处**：原版用 **public 裸 `UnityAction` 字段**，外部能 `channel.OnEventRaised = null` 抹掉所有订阅、也能越权 `Invoke`。本仿写故意改用 `event` 关键字封死这点（只能 `+=`/`-=`）。这是**有意偏离原版**以演示安全边界；若要 1:1 复刻原版的「灵活但不安全」，把 `event` 去掉即可。另一处易错：Listener 漏调 `Detach`/`Dispose` 会让通道悬空持有已销毁对象——对应原版漏写 `OnDisable -=` 的 `MissingReferenceException`。
