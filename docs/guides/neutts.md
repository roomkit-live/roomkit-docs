# NeuTTS Provider

RoomKit integrates with [NeuTTS](https://github.com/neuphonic/neutts) for LLM-based text-to-speech with zero-shot voice cloning. NeuTTS uses a Qwen2.5 backbone with NeuCodec to produce high-quality 24kHz audio, with native streaming support via GGUF quantized models.

```bash
pip install neutts
```

> **Note:** NeuTTS requires `espeak-ng` as a system dependency for phonemization.
> Install it with `sudo apt install espeak-ng` (Ubuntu) or `brew install espeak-ng` (macOS).

---

## Key features

- **Zero-shot voice cloning** — clone any voice from 3-15 seconds of reference audio
- **Native streaming** — GGUF models generate audio in ~0.5s chunks as they decode
- **CPU-friendly** — GGUF Q8 quantization runs in real-time on laptop-class CPUs
- **French-first** — `neutts-nano-french` optimized for French, with other languages available in the [multilingual collection](https://huggingface.co/collections/neuphonic/neutts-nano-multilingual-collection)

---

## Quick start

```python
from roomkit.voice.tts.neutts import NeuTTSConfig, NeuTTSProvider, NeuTTSVoiceConfig

tts = NeuTTSProvider(NeuTTSConfig(
    voices={
        "marie": NeuTTSVoiceConfig(
            ref_audio="reference.wav",
            ref_text="Bonjour, je suis Marie.",
        ),
    },
))

await tts.warmup()

# Full synthesis
audio = await tts.synthesize("Bonjour le monde", voice="marie")

# Streaming synthesis
async for chunk in tts.synthesize_stream("Bonjour le monde", voice="marie"):
    process(chunk)  # AudioChunk with PCM S16LE at 24kHz
```

---

## Model variants

NeuTTS Nano models are available in several quantizations:

| Model | HuggingFace repo | Size | Speed | Quality |
|---|---|---|---|---|
| **French Q8 GGUF** (default) | `neuphonic/neutts-nano-french-q8-gguf` | ~228M params | Real-time on CPU | Best |
| French Q4 GGUF | `neuphonic/neutts-nano-french-q4-gguf` | ~228M params | Faster | Good |
| French (PyTorch) | `neuphonic/neutts-nano-french` | ~228M params | Requires GPU | Best |

> **Important:** Streaming (`synthesize_stream()`) is only available with GGUF models.
> The PyTorch backbone does not support streaming and will raise `NotImplementedError`.

Other languages are available in the [NeuTTS Nano Multilingual Collection](https://huggingface.co/collections/neuphonic/neutts-nano-multilingual-collection). Set `backbone_repo` to the desired model.

---

## Configuration

All parameters are set via `NeuTTSConfig`:

| Parameter | Default | Description |
|---|---|---|
| `backbone_repo` | `"neuphonic/neutts-nano-french-q8-gguf"` | HuggingFace repo ID or local path to the GGUF backbone model. |
| `codec_repo` | `"neuphonic/neucodec"` | HuggingFace repo ID or local path to the NeuCodec model. |
| `device` | `"cpu"` | Inference device: `"cpu"` or `"cuda"`. |
| `voices` | `{}` | Named voice clones — each maps a name to a `NeuTTSVoiceConfig`. |
| `temperature` | `1.0` | Sampling temperature. |
| `top_k` | `50` | Top-k sampling. |
| `max_length` | `2048` | Maximum generation length in tokens. |

### Voice configuration

Each voice is defined by `NeuTTSVoiceConfig`:

| Parameter | Description |
|---|---|
| `ref_audio` | Path to reference WAV file (3-15s of clean speech, mono, 16-44 kHz). |
| `ref_text` | Transcript of the reference audio. |

> **Best practice:** Use a same-language reference audio for best cloning quality.
> For the French model, provide a French reference clip.

---

## Reference audio guidelines

For optimal voice cloning results:

- **Duration**: 3-15 seconds of continuous speech
- **Format**: WAV file, mono channel
- **Sample rate**: 16-44 kHz
- **Quality**: Clean recording with minimal background noise
- **Content**: Natural, continuous speech (like a monologue) with few pauses

---

## How it works

### Warmup

`warmup()` runs two steps in background threads:

1. **Model loading** — downloads (if needed) and initializes the GGUF backbone + NeuCodec decoder
2. **Reference encoding** — pre-encodes all configured voice references into `ref_codes`

Both steps are CPU-heavy, so they run via `asyncio.to_thread()` to avoid blocking the event loop.

### Synthesis

`synthesize()` calls `tts.infer()` in a thread, producing a numpy float32 array at 24kHz. The array is converted to PCM S16LE, wrapped in a WAV header, and returned as a base64 data URL in `AudioContent`.

### Streaming

`synthesize_stream()` uses `tts.infer_stream()`, which yields numpy chunks as they're decoded by the GGUF backbone. Each chunk (~0.5s of audio) is converted to PCM S16LE and yielded as an `AudioChunk`. A queue bridges the blocking generator to the async world.

---

## Local models

For offline or air-gapped deployments, download the models locally:

```bash
# Download GGUF backbone
huggingface-cli download neuphonic/neutts-nano-french-q8-gguf \
    --local-dir /path/to/models/neutts-nano-french/

# Download NeuCodec (shared across all NeuTTS models)
huggingface-cli download neuphonic/neucodec \
    --local-dir /path/to/models/neucodec/
```

Then point the config to local paths:

```python
tts = NeuTTSProvider(NeuTTSConfig(
    backbone_repo="/path/to/models/neutts-nano-french",
    codec_repo="/path/to/models/neucodec",
    voices={"marie": NeuTTSVoiceConfig(ref_audio="ref.wav", ref_text="...")},
))
```

---

## Lazy loader

If you want to avoid importing `neutts` at module level:

```python
from roomkit.voice import get_neutts_provider, get_neutts_config, get_neutts_voice_config

NeuTTSProvider = get_neutts_provider()
NeuTTSConfig = get_neutts_config()
NeuTTSVoiceConfig = get_neutts_voice_config()

tts = NeuTTSProvider(NeuTTSConfig(
    voices={"marie": NeuTTSVoiceConfig(ref_audio="ref.wav", ref_text="...")},
))
```

---

## Full pipeline example

```python
from roomkit import VoiceChannel
from roomkit.voice.backends.local import LocalAudioBackend
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.vad.sherpa_onnx import SherpaOnnxVADConfig, SherpaOnnxVADProvider
from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider
from roomkit.voice.tts.neutts import NeuTTSConfig, NeuTTSProvider, NeuTTSVoiceConfig

# VAD
vad = SherpaOnnxVADProvider(
    SherpaOnnxVADConfig(model="ten-vad.onnx")
)

# STT
stt = SherpaOnnxSTTProvider(
    SherpaOnnxSTTConfig(
        mode="transducer",
        encoder="encoder.onnx",
        decoder="decoder.onnx",
        joiner="joiner.onnx",
        tokens="tokens.txt",
    )
)

# TTS — NeuTTS with French voice clone
tts = NeuTTSProvider(NeuTTSConfig(
    voices={
        "default": NeuTTSVoiceConfig(
            ref_audio="reference_fr.wav",
            ref_text="Bonjour, je suis un assistant vocal.",
        ),
    },
))

# Backend — local mic/speaker (16kHz input, 24kHz output for NeuTTS)
backend = LocalAudioBackend(
    input_sample_rate=16000,
    output_sample_rate=24000,
)

# Wire it together
voice = VoiceChannel(
    "voice",
    stt=stt,
    tts=tts,
    backend=backend,
    pipeline=AudioPipelineConfig(vad=vad),
)
```

---

## Examples

- [`examples/voice_neutts.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_neutts.py) — complete local voice assistant with NeuTTS French voice cloning + sherpa-onnx STT/VAD + Ollama LLM.
