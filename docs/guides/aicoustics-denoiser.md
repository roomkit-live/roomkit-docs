# ai|coustics Denoiser

The `AICousticsDenoiserProvider` integrates **ai|coustics Quail** speech enhancement models into the RoomKit audio pipeline. Quail provides neural noise suppression, dereverberation, and speaker isolation (Voice Focus), optimized for STT/ASR accuracy.

Key characteristics:

- **On-device** — runs locally via the `aic-sdk` Python package (Rust + PyO3)
- **CPU-only** — no GPU required
- **~2 ms inference** per 10 ms frame
- **30 ms algorithmic delay**

## Installation

```bash
pip install roomkit[aicoustics]
```

Set your license key as an environment variable:

```bash
export AIC_SDK_LICENSE="your-license-key"
```

Or pass it directly in the config:

```python
config = AICousticsDenoiserConfig(license_key="your-license-key")
```

## Quick Start

```python
from roomkit.voice.pipeline.denoiser import (
    AICousticsDenoiserConfig,
    AICousticsDenoiserProvider,
)
from roomkit.voice.pipeline import AudioPipelineConfig

denoiser = AICousticsDenoiserProvider(
    AICousticsDenoiserConfig(model="quail-vf-2.0-l-16khz")
)
pipeline = AudioPipelineConfig(denoiser=denoiser)
```

## Model Comparison

| Model | Size | Sample Rate | Features | Use Case |
|-------|------|-------------|----------|----------|
| `quail-vf-2.0-l-16khz` | Large | 16 kHz | Denoise + dereverb + Voice Focus | Best quality for voice AI / STT |
| `quail-2.0-l-16khz` | Large | 16 kHz | Denoise + dereverb | General enhancement without speaker isolation |
| `quail-2.0-s-16khz` | Small | 16 kHz | Denoise + dereverb | Lower latency, smaller footprint |

The **VF (Voice Focus)** models include speaker isolation — they suppress competing speakers and keep only the primary talker. This is especially useful for multi-party environments.

## Configuration Reference

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | `str` | `"quail-vf-2.0-l-16khz"` | Model identifier for download |
| `model_dir` | `str` | `"./models"` | Local cache directory for downloaded models |
| `license_key` | `str` | `""` | SDK license key (falls back to `AIC_SDK_LICENSE` env var) |
| `enhancement_level` | `float` | `0.8` | Enhancement strength from 0.0 to 1.0 |
| `num_channels` | `int` | `1` | Audio channels (1 = mono, 2 = stereo) |

## Enhancement Level Tuning

The `enhancement_level` parameter controls the aggressiveness of noise removal:

| Level | Style | Best For |
|-------|-------|----------|
| `0.3–0.5` | Conservative | Mild background noise, preserves natural ambiance |
| `0.8` | Balanced | **Recommended** — best WER for voice AI workloads |
| `1.0` | Aggressive | Heavy noise environments, may introduce artifacts |

```python
# Conservative — light touch
config = AICousticsDenoiserConfig(enhancement_level=0.5)

# Balanced — best for STT accuracy (default)
config = AICousticsDenoiserConfig(enhancement_level=0.8)

# Aggressive — noisy environments
config = AICousticsDenoiserConfig(enhancement_level=1.0)
```

## Full Voice Pipeline Integration

```python
from roomkit import RoomKit, VoiceChannel
from roomkit.voice.backends import LocalAudioBackend
from roomkit.voice.stt import DeepgramSTTProvider
from roomkit.voice.tts import ElevenLabsTTSProvider
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.denoiser import (
    AICousticsDenoiserConfig,
    AICousticsDenoiserProvider,
)
from roomkit.voice.pipeline.vad import SherpaOnnxVADProvider

kit = RoomKit()

denoiser = AICousticsDenoiserProvider(
    AICousticsDenoiserConfig(
        model="quail-vf-2.0-l-16khz",
        enhancement_level=0.8,
    )
)

pipeline = AudioPipelineConfig(
    denoiser=denoiser,
    vad=SherpaOnnxVADProvider(model_path="silero_vad.onnx"),
)

voice = VoiceChannel(
    "voice",
    backend=LocalAudioBackend(),
    stt=DeepgramSTTProvider(...),
    tts=ElevenLabsTTSProvider(...),
    pipeline_config=pipeline,
)

kit.register_channel(voice)
```

## Performance Notes

- **Inference latency**: ~2 ms per 10 ms frame on modern CPUs
- **Algorithmic delay**: 30 ms (3 frames at 10 ms)
- **CPU-only**: no GPU required; uses Rust backend via PyO3
- **Model download**: first call triggers a one-time model download to `model_dir`
- **Thread-safe**: internal locking allows safe concurrent access

The provider buffers incoming audio to match Quail's expected frame size. If your pipeline frame size doesn't match exactly, the provider handles buffering internally — no configuration needed.
