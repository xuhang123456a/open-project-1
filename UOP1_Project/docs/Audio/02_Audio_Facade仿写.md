# Audio Facade 仿写

## 设计映射表

| 原实现 | 精简版 | 标注 |
|--------|--------|------|
| `AudioManager` 音频管理器 | 保留核心逻辑 | ✅保留 |
| `AudioCue` 音频触发器 | 保留核心逻辑 | ✅保留 |
| `SoundEmitterPoolSO` 池 | 简化为对象池 | ⚠️ 简化 |
| `AudioCueKey` 控制键 | 保留 | ✅保留 |
| `AudioMixer` 音量控制 | 简化 | ⚠️ 简化 |

---

## 最小复刻代码（~120行）

```csharp
using System.Collections.Generic;
using UnityEngine;

// ==================== 音频配置 ====================

[CreateAssetMenu(menuName = "Audio/Audio Cue")]
public class AudioCueSO : ScriptableObject
{
    public AudioClip[] clips;
    public float volume = 1f;
    public float pitch = 1f;
    public bool loop;
}

[CreateAssetMenu(menuName = "Audio/Audio Config")]
public class AudioConfigurationSO : ScriptableObject
{
    public AudioMixerGroup mixerGroup;
    public float spatialBlend = 1f;
}

// ==================== 控制键 ====================

public struct AudioCueKey
{
    public int Id;
    public bool IsValid => Id != -1;
    public static AudioCueKey Invalid => new AudioCueKey { Id = -1 };
}

// ==================== 音频管理器 ====================

public class AudioManager : MonoBehaviour
{
    [SerializeField] private int _initialSize = 10;
    [SerializeField] private AudioMixer _audioMixer;
    
    private Queue<AudioSource> _availableSources;
    private Dictionary<int, AudioSource> _activeSources;
    private int _nextId;
    
    private void Awake()
    {
        _availableSources = new Queue<AudioSource>();
        _activeSources = new Dictionary<int, AudioSource>();
        
        // 预热
        for (int i = 0; i < _initialSize; i++)
        {
            var source = gameObject.AddComponent<AudioSource>();
            source.playOnAwake = false;
            _availableSources.Enqueue(source);
        }
    }
    
    public AudioCueKey PlaySound(AudioCueSO cue, AudioConfigurationSO config, Vector3 position)
    {
        if (_availableSources.Count == 0)
        {
            var newSource = gameObject.AddComponent<AudioSource>();
            _availableSources.Enqueue(newSource);
        }
        
        var source = _availableSources.Dequeue();
        source.transform.position = position;
        source.clip = cue.clips[Random.Range(0, cue.clips.Length)];
        source.volume = cue.volume;
        source.pitch = cue.pitch;
        source.loop = cue.loop;
        source.spatialBlend = config.spatialBlend;
        source.Play();
        
        var key = new AudioCueKey { Id = _nextId++ };
        _activeSources[key.Id] = source;
        
        if (!cue.loop)
        {
            StartCoroutine(ReturnWhenFinished(key, source.clip.length));
        }
        
        return key;
    }
    
    public void StopSound(AudioCueKey key)
    {
        if (_activeSources.TryGetValue(key.Id, out var source))
        {
            source.Stop();
            _activeSources.Remove(key.Id);
            _availableSources.Enqueue(source);
        }
    }
    
    public void SetVolume(string parameter, float value)
    {
        // 转换为分贝
        float dB = value > 0 ? 20f * Mathf.Log10(value) : -80f;
        _audioMixer.SetFloat(parameter, dB);
    }
    
    private System.Collections.IEnumerator ReturnWhenFinished(AudioCueKey key, float delay)
    {
        yield return new WaitForSeconds(delay);
        StopSound(key);
    }
}

// ==================== 音频触发器 ====================

public class AudioCue : MonoBehaviour
{
    [SerializeField] private AudioCueSO _cue;
    [SerializeField] private AudioConfigurationSO _config;
    [SerializeField] private bool _playOnStart;
    
    private AudioCueKey _key = AudioCueKey.Invalid;
    
    private void Start()
    {
        if (_playOnStart) Play();
    }
    
    public void Play()
    {
        _key = FindObjectOfType<AudioManager>().PlaySound(_cue, _config, transform.position);
    }
    
    public void Stop()
    {
        if (_key.IsValid)
        {
            FindObjectOfType<AudioManager>().StopSound(_key);
            _key = AudioCueKey.Invalid;
        }
    }
    
    private void OnDisable() => Stop();
}
```

---

## 取舍自检

| 项目 | 标注 | 说明 |
|------|------|------|
| 对象池 | ✅保留 | 核心机制 |
| 控制键 | ✅保留 | 核心机制 |
| 音量分层 | ⚠️ 简化 | 只保留基础逻辑 |
| AudioMixer | ⚠️ 简化 | 只保留接口 |
| **最容易搞错** | ⚠️ | 忘记在OnDisable中停止音频 |
