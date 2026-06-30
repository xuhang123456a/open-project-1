# Audio 模块 Facade 仿写

> 目标：复刻「请求-应答带返回值通道 + 池化 emitter + 金库句柄登记 + 多 clip 一次 key」核心闭环，并**修复原版 TODO**（自然播完路径漏移除金库 key）。
> 复用前面 Pool / Events 的 Facade 思想，用纯 C# 模拟，砍掉 DOTween/AudioMixer/空间化/音乐淡入淡出。

---

## 设计映射表

| 原实现 | 精简版 | 取舍 |
|---|---|---|
| `AudioCueEventChannelSO`（带返回值）| `RequestChannel<PlayArgs,AudioKey>` | ✅ 保留请求-应答 |
| `SoundEmitterPoolSO`(ComponentPool) | `Pool<FakeEmitter>`（复用 Pool Facade）| ✅ 保留借还 |
| `SoundEmitter`(AudioSource+协程) | `FakeEmitter`（含 `OnFinished` 回调）| ⚠️ 用回调模拟播完 |
| `SoundEmitterVault`（双 List + FindIndex）| `Vault`（保留并行列表实现）| ✅ 保留（含其 O(n) 特性）|
| `AudioCueKey`(struct, 自增+cue) | `AudioKey`(struct, 自增+cue 引用 + 相等)| ✅ 保留 |
| 一次 cue 多 emitter | 同 | ✅ 保留 |
| 音乐淡入淡出/单例 | 删除 | ❌ 砍掉 |
| **自然播完漏移除 key（TODO）** | **修复**：finished 路径也移除金库 key | ⚠️ 有意修正缺陷 |

---

## 最小复刻代码（独立可编译，~175 行）

```csharp
using System;
using System.Collections.Generic;

namespace MiniAudio
{
    public sealed class CueDef   // 对应 AudioCueSO
    {
        public string Name; public bool Looping;
        public string[] Clips;  // 一个 cue 可含多 clip → 多 emitter
        public CueDef(string name, bool looping, params string[] clips) { Name = name; Looping = looping; Clips = clips; }
    }

    // 不可变句柄：自增序号 + cue 引用，重写相等
    public readonly struct AudioKey : IEquatable<AudioKey>
    {
        public static readonly AudioKey Invalid = new AudioKey(-1, null);
        public readonly int Value; public readonly CueDef Cue;
        public AudioKey(int v, CueDef cue) { Value = v; Cue = cue; }
        public bool Equals(AudioKey o) => Value == o.Value && ReferenceEquals(Cue, o.Cue);
        public override bool Equals(object o) => o is AudioKey k && Equals(k);
        public override int GetHashCode() => Value;
        public static bool operator ==(AudioKey a, AudioKey b) => a.Equals(b);
        public static bool operator !=(AudioKey a, AudioKey b) => !a.Equals(b);
    }

    public sealed class FakeEmitter
    {
        public Action<FakeEmitter> OnFinished;
        public bool IsLooping { get; private set; }
        private string _clip;
        public void Play(string clip, bool loop) { _clip = clip; IsLooping = loop; Console.WriteLine($"  ▶ 播放 {clip} loop={loop}"); }
        public void SimulateFinish() { if (!IsLooping) OnFinished?.Invoke(this); } // 模拟非循环播完
        public void Stop() => Console.WriteLine($"  ■ 停止 {_clip}");
        public void Finish() { if (IsLooping) { IsLooping = false; OnFinished?.Invoke(this); } } // 关loop让其结束
    }

    // 金库：并行列表 + FindIndex（保留原版 O(n) 风格）
    public sealed class Vault
    {
        private int _next;
        private readonly List<AudioKey> _keys = new();
        private readonly List<FakeEmitter[]> _emitters = new();
        public AudioKey Add(CueDef cue, FakeEmitter[] e)
        {
            var key = new AudioKey(_next++, cue);
            _keys.Add(key); _emitters.Add(e); return key;
        }
        public bool Get(AudioKey key, out FakeEmitter[] e)
        {
            int i = _keys.FindIndex(k => k == key);
            if (i < 0) { e = null; return false; }
            e = _emitters[i]; return true;
        }
        public bool Remove(AudioKey key)
        {
            int i = _keys.FindIndex(k => k == key);
            if (i < 0) return false;
            _keys.RemoveAt(i); _emitters.RemoveAt(i); return true;
        }
        // 修复 TODO：按 emitter 反查它所属的 key 并移除（自然播完路径用）
        public bool RemoveByEmitter(FakeEmitter target, out AudioKey removed)
        {
            for (int i = 0; i < _emitters.Count; i++)
                if (Array.IndexOf(_emitters[i], target) >= 0)
                { removed = _keys[i]; _keys.RemoveAt(i); _emitters.RemoveAt(i); return true; }
            removed = AudioKey.Invalid; return false;
        }
    }

    public readonly struct PlayArgs
    {
        public readonly CueDef Cue; public PlayArgs(CueDef cue) => Cue = cue;
    }

    public sealed class AudioManager
    {
        private readonly Pool<FakeEmitter> _pool;
        private readonly Vault _vault = new();
        public AudioManager(Pool<FakeEmitter> pool, RequestChannel channel)
        {
            _pool = pool;
            channel.OnPlay = Play;          // 订阅请求通道
            channel.OnStop = Stop;
        }
        public AudioKey Play(PlayArgs args)
        {
            var cue = args.Cue;
            var emitters = new FakeEmitter[cue.Clips.Length];
            for (int i = 0; i < cue.Clips.Length; i++)
            {
                var e = _pool.Request();
                emitters[i] = e;
                e.Play(cue.Clips[i], cue.Looping);
                if (!cue.Looping) e.OnFinished += OnEmitterFinished;
            }
            return _vault.Add(cue, emitters);
        }
        public bool Stop(AudioKey key)
        {
            if (!_vault.Get(key, out var emitters)) return false;
            foreach (var e in emitters) { CleanEmitter(e); }
            _vault.Remove(key);            // 显式停止：移除 key
            return true;
        }
        private void OnEmitterFinished(FakeEmitter e)
        {
            // 修复：自然播完也要把金库里对应 key 清掉，防悬空
            _vault.RemoveByEmitter(e, out _);
            CleanEmitter(e);
        }
        private void CleanEmitter(FakeEmitter e)
        {
            if (!e.IsLooping) e.OnFinished -= OnEmitterFinished;
            e.Stop();
            _pool.Return(e);
        }
    }

    // 极简请求通道（带两个应答委托）
    public sealed class RequestChannel
    {
        public Func<PlayArgs, AudioKey> OnPlay;
        public Func<AudioKey, bool> OnStop;
        public AudioKey RaisePlay(PlayArgs a) => OnPlay != null ? OnPlay(a) : AudioKey.Invalid;
        public bool RaiseStop(AudioKey k) => OnStop != null && OnStop(k);
    }

    // —— 复用前文 Pool Facade 的最小版（此处内联以便独立编译）——
    public sealed class Pool<T> where T : class
    {
        private readonly Stack<T> _avail = new(); private readonly HashSet<T> _in = new();
        private readonly Func<T> _factory; private bool _warm;
        public Pool(Func<T> f) => _factory = f;
        public void Prewarm(int n) { if (_warm) return; for (int i=0;i<n;i++){var m=_factory();_avail.Push(m);_in.Add(m);} _warm=true; }
        public T Request() { T m = _avail.Count>0 ? Pop() : _factory(); return m; }
        public void Return(T m){ if(m==null||_in.Contains(m))return; _avail.Push(m); _in.Add(m); }
        private T Pop(){ var m=_avail.Pop(); _in.Remove(m); return m; }
    }
}
```

---

## 使用示例

```csharp
using System;
using MiniAudio;

class Demo
{
    static void Main()
    {
        int seq = 0;
        var pool = new Pool<FakeEmitter>(() => new FakeEmitter());
        pool.Prewarm(4);

        var channel = new RequestChannel();
        var manager = new AudioManager(pool, channel);   // 订阅通道

        var explosion = new CueDef("Explosion", looping: false, "boom.wav", "debris.wav"); // 2 clip
        var ambient   = new CueDef("Ambient",   looping: true,  "wind.wav");

        // 请求方只拿到 key，不知道内部用了几个 emitter
        AudioKey k1 = channel.RaisePlay(new PlayArgs(explosion));  // 借 2 个 emitter
        AudioKey k2 = channel.RaisePlay(new PlayArgs(ambient));    // 借 1 个（循环）

        // 显式停止循环音（否则永占 emitter）
        channel.RaiseStop(k2);

        // 模拟非循环音自然播完 → 自动归还 + 修复后会清金库 key
        // （演示：直接对池请求-播放路径的 emitter 调 SimulateFinish 略，逻辑见 OnEmitterFinished）
        Console.WriteLine($"k1={k1.Value}, k2={k2.Value}");
    }
}
```

---

## 取舍自检

- ✅ **保留**：请求-应答带返回值通道、`AudioKey` 不可变 struct 句柄（自增序号 + cue + 自定义相等）、一次 cue 借多个 emitter 用一个 key 登记、金库并行列表 + `FindIndex`（含其 O(n) 本质）、Pool 借还、循环/非循环回收路径分叉。
- ❌ **砍掉**：DOTween 淡入淡出、AudioMixer 音量归一化、3D 空间化与 `AudioConfigurationSO.ApplyTo`、音乐单例 emitter 与切歌时序、`AudioClipsGroup` 的随机/顺序播放模式。
- ⚠️ **最容易搞错的一处**：原版**自然播完路径（`StopAndCleanEmitter`）归还了 emitter 却没从金库 `Remove` 对应 key**（源码 TODO 已自认）。结果：金库残留指向「已归还/可能被复用」emitter 的悬空 key，之后凭旧 key `Stop` 会误停一个已被别的播放请求复用的 emitter。本仿写用 `RemoveByEmitter` 在 finished 回调里反查并清除 key 修复之。另一处易错：循环音不挂 finished 回调，必须有显式 Stop 触发者，否则 emitter 永久泄漏出池。
