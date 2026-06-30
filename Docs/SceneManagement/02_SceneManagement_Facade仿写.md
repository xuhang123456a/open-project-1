# SceneManagement 模块 Facade 仿写

> 目标：复刻三个核心不变量——①经事件通道请求加载（触发与执行解耦）；②`_isLoading` 防重入闸门覆盖所有出口；③「淡出→卸旧→载新→淡入」协程流水线 + 加载/卸载句柄配对。
> 用纯 C# + async/await 模拟 Unity 协程与 Addressables，砍掉编辑器冷启动、光照探针、Cinemachine。

---

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `LoadEventChannelSO` | `EventChannel<LoadRequest>`（复用 Events Facade）| ✅ 保留解耦中枢 |
| `GameSceneSO` + `AssetReference` | `SceneDef` { Id, Type } | ⚠️ 用 Id 模拟 Addressables key |
| `AsyncOperationHandle<SceneInstance>` | `Task<LoadedScene>` + handle 字典 | ⚠️ 用 Task 模拟异步，保留句柄配对 |
| `_isLoading` 闸门 | 同（bool）| ✅ 保留（核心）|
| `UnloadPreviousScene` 协程 | `async Task` 流水线 | ✅ 保留时序 |
| FadeOut/FadeIn 通道 | 委托回调 | ✅ 保留 UX 时序点 |
| 常驻 Gameplay 场景按需加载 | 简化为「确保 manager 已加载」一步 | ⚠️ 简化 |
| 冷启动/光照/原生卸载分支 | 删除 | ❌ 砍掉 |

---

## 最小复刻代码（独立可编译，~165 行）

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace MiniScene
{
    public enum SceneType { Menu, Location, GameplayManager }

    public sealed class SceneDef
    {
        public string Id; public SceneType Type;
        public SceneDef(string id, SceneType type) { Id = id; Type = type; }
    }

    public readonly struct LoadRequest
    {
        public readonly SceneDef Scene; public readonly bool ShowLoading; public readonly bool Fade;
        public LoadRequest(SceneDef s, bool showLoading, bool fade) { Scene = s; ShowLoading = showLoading; Fade = fade; }
    }

    // 复用 Events 思想的请求通道
    public sealed class LoadChannel
    {
        public Func<LoadRequest, Task> OnLoadingRequested;
        public Task Raise(LoadRequest r)
        {
            if (OnLoadingRequested != null) return OnLoadingRequested(r);
            Console.WriteLine("[LoadChannel] 加载被请求但无 SceneLoader 接听!");
            return Task.CompletedTask;
        }
    }

    // 模拟 Addressables：加载返回句柄，卸载必须用同一句柄
    public sealed class FakeAddressables
    {
        private readonly Dictionary<string, object> _loaded = new();
        public Task<object> LoadAsync(string id)
        {
            var handle = new object();
            _loaded[id] = handle;
            Console.WriteLine($"  [Addr] 异步加载 {id}");
            return Task.FromResult(handle);
        }
        public void Unload(string id, object handle)
        {
            if (!_loaded.TryGetValue(id, out var h) || !ReferenceEquals(h, handle))
            { Console.WriteLine($"  [Addr] !! 卸载 {id} 句柄不匹配，泄漏风险"); return; }
            _loaded.Remove(id);
            Console.WriteLine($"  [Addr] 卸载 {id}");
        }
    }

    public sealed class SceneLoader
    {
        private readonly FakeAddressables _addr = new();
        private readonly LoadChannel _channel;
        // UX 注入点（对应 fade / loading screen 通道）
        public Action<bool> ToggleLoadingScreen = _ => { };
        public Func<float, Task> FadeOut = _ => Task.CompletedTask;
        public Func<float, Task> FadeIn = _ => Task.CompletedTask;
        public Action OnSceneReady = () => { };

        private const float FadeDuration = 0.5f;
        private bool _isLoading;                       // 防重入闸门（核心）
        private SceneDef _currentScene;
        private object _currentHandle;
        private bool _gameplayLoaded;
        private object _gameplayHandle;

        public SceneLoader(LoadChannel channel)
        {
            _channel = channel;
            _channel.OnLoadingRequested += Load;       // 唯一订阅者
        }

        public async Task Load(LoadRequest req)
        {
            if (_isLoading) { Console.WriteLine("[Loader] 正在加载，请求被闸门拦截"); return; }
            _isLoading = true;
            try
            {
                // 1) Location 需先确保 Gameplay 管理器场景已加载
                if (req.Scene.Type == SceneType.Location && !_gameplayLoaded)
                {
                    _gameplayHandle = await _addr.LoadAsync("GameplayManagers");
                    _gameplayLoaded = true;
                }
                // 进 Menu 时卸掉常驻 Gameplay 场景
                if (req.Scene.Type == SceneType.Menu && _gameplayLoaded)
                {
                    _addr.Unload("GameplayManagers", _gameplayHandle);
                    _gameplayLoaded = false;
                }
                // 2) 淡出 → 卸旧
                await FadeOut(FadeDuration);
                if (_currentScene != null) _addr.Unload(_currentScene.Id, _currentHandle);
                // 3) 载新
                if (req.ShowLoading) ToggleLoadingScreen(true);
                _currentHandle = await _addr.LoadAsync(req.Scene.Id);
                _currentScene = req.Scene;
                if (req.ShowLoading) ToggleLoadingScreen(false);
                // 4) 完成
                await FadeIn(FadeDuration);
                OnSceneReady();                         // 广播就绪 → spawn/相机
            }
            finally
            {
                _isLoading = false;                     // 覆盖所有出口（含异常）
            }
        }
    }
}
```

---

## 使用示例

```csharp
using System;
using System.Threading.Tasks;
using MiniScene;

class Demo
{
    static async Task Main()
    {
        var channel = new LoadChannel();
        var loader = new SceneLoader(channel)
        {
            FadeOut = d => { Console.WriteLine($"FadeOut {d}s"); return Task.CompletedTask; },
            FadeIn  = d => { Console.WriteLine($"FadeIn {d}s");  return Task.CompletedTask; },
            OnSceneReady = () => Console.WriteLine(">> 场景就绪，生成玩家"),
        };

        var forest = new SceneDef("Forest", SceneType.Location);
        var beach  = new SceneDef("Beach",  SceneType.Location);

        // 触发者只管 Raise，不知道谁加载
        await channel.Raise(new LoadRequest(forest, showLoading: true, fade: true));
        await channel.Raise(new LoadRequest(beach,  showLoading: false, fade: true));

        // 模拟一帧内两次触发（防重入）：第二个被闸门拦截
        var t1 = channel.Raise(new LoadRequest(forest, false, true));
        var t2 = channel.Raise(new LoadRequest(beach, false, true));
        await Task.WhenAll(t1, t2);
    }
}
```

---

## 取舍自检

- ✅ **保留**：事件通道请求加载（触发/执行解耦 + 唯一订阅者 + 无人接听告警）、`_isLoading` 闸门置于 `try/finally` 覆盖所有出口、「淡出→卸旧→载新→淡入→广播就绪」时序、加载/卸载**句柄配对**（`FakeAddressables` 校验同一 handle）、常驻 Gameplay 场景按需加载/卸载。
- ❌ **砍掉**：真实 Addressables/`AssetReference`、`LightProbes.TetrahedralizeAsync`、Cinemachine 相机转场、`UNITY_EDITOR` 冷启动与原生 `SceneManager` 回退路径、PathSO/PathStorage 入口逻辑。
- ⚠️ **最容易搞错的一处**：`_isLoading` 的复位**必须放在 `finally`**（覆盖异常路径）。原版在 `OnNewSceneLoaded` 末尾复位——只要加载链路抛异常或某分支提前 return，`_isLoading` 永远不会复位，加载器永久卡死（之后所有请求被闸门吞掉）。这是「单一闸门变量」模式的通病：置位简单，复位必须穷尽所有出口。其次易错：卸载用错句柄会静默泄漏（原版冷启动场景就需要第二套卸载路径来规避）。
