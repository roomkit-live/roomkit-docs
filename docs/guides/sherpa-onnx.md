# sherpa-onnx Providers

RoomKit integrates with [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) for neural audio processing. All sherpa-onnx providers are pure-Python — no system libraries required.

```bash
pip install roomkit[sherpa-onnx]
```

---

## GPU Acceleration (CUDA)

All sherpa-onnx providers support NVIDIA GPU acceleration via CUDA for faster inference.

> **Important:** The default `sherpa-onnx` pip package is **CPU-only**. You must install
> the CUDA-specific wheel to enable GPU acceleration.

### Prerequisites

- NVIDIA GPU with compute capability 6.0+
- NVIDIA driver (verify with `nvidia-smi`)
- **cuDNN 9** system library (see installation below)

### Step 1 — Install cuDNN 9 (system-wide)

sherpa-onnx CUDA wheels link against cuDNN 9. This is a system library, not a Python package.

**Ubuntu/Debian:**

```bash
# Add NVIDIA package repository (if not already configured)
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update

# Install cuDNN 9 for CUDA 12
sudo apt-get -y install cudnn9-cuda-12
```

For Ubuntu 22.04, replace `ubuntu2404` with `ubuntu2204` in the URL above.

Verify:

```bash
ldconfig -p | grep cudnn   # should list libcudnn.so.9
```

### Step 2 — Install the sherpa-onnx CUDA wheel

The CUDA wheels are hosted on a separate index (not on PyPI).

**With uv (recommended):**

```bash
uv pip install sherpa-onnx==1.12.23+cuda12.cudnn9 \
    -f https://k2-fsa.github.io/sherpa/onnx/cuda.html
```

**With pip:**

```bash
pip install sherpa-onnx==1.12.23+cuda12.cudnn9 --no-index \
    -f https://k2-fsa.github.io/sherpa/onnx/cuda.html
```

> **CUDA 11 vs 12:** If your system has CUDA 11.x, use `sherpa-onnx==1.12.23+cuda`
> instead. Check with `nvcc --version`.

### Step 3 — Verify

```bash
python -c "
import sherpa_onnx
print('sherpa-onnx version:', sherpa_onnx.__version__)
print('file:', sherpa_onnx.__file__)
"
```

When you set `provider="cuda"` in a config and CUDA is available, you should see
GPU memory usage increase in `nvidia-smi`.

### Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `libcublasLt.so.11: cannot open shared object file` | sherpa-onnx CUDA 11 wheel installed on CUDA 12 system | Install the `+cuda12.cudnn9` wheel instead |
| `libcublasLt.so.12: cannot open shared object file` | CUDA toolkit not installed | `sudo apt install cuda-toolkit-12` |
| `libonnxruntime_providers_cuda.so: Failed to load` | cuDNN not installed | Install cuDNN 9 (Step 1 above) |
| `libcudnn.so.9: cannot open shared object file` | cuDNN 9 missing | `sudo apt install cudnn9-cuda-12` |
| `Please compile with -DSHERPA_ONNX_ENABLE_GPU=ON` | CPU-only sherpa-onnx wheel installed | Install the CUDA wheel (Step 2 above) |
| CUDA requested but falls back to CPU silently | Wrong wheel or missing cuDNN | Check `nvidia-smi` for GPU memory usage |

### Configuration

Set `provider="cuda"` on any sherpa-onnx config dataclass:

| Provider | Config class | Recommended provider | Why |
|---|---|---|---|
| STT | `SherpaOnnxSTTConfig` | `"cuda"` | Batch inference on larger models — benefits from GPU |
| TTS | `SherpaOnnxTTSConfig` | `"cuda"` | Synthesis is compute-heavy — benefits from GPU |
| VAD | `SherpaOnnxVADConfig` | `"cpu"` | Runs per-frame (every 20ms), tiny model — GPU transfer overhead makes CUDA slower |
| Denoiser | `SherpaOnnxDenoiserConfig` | `"cpu"` | Runs per-frame (every 20ms), tiny model — GPU transfer overhead makes CUDA slower |

> **Best practice:** Use CUDA for STT and TTS only. VAD and denoiser process every
> 20ms audio frame with tiny models (2-5MB). The GPU memory transfer overhead per
> frame exceeds the computation time, making CUDA slower than CPU for these providers.
> Worse, it can cause 100% CPU usage and system hangs.

### Example

```python
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.denoiser.sherpa_onnx import (
    SherpaOnnxDenoiserConfig,
    SherpaOnnxDenoiserProvider,
)
from roomkit.voice.pipeline.vad.sherpa_onnx import (
    SherpaOnnxVADConfig,
    SherpaOnnxVADProvider,
)
from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider
from roomkit.voice.tts.sherpa_onnx import SherpaOnnxTTSConfig, SherpaOnnxTTSProvider

# STT + TTS on GPU, VAD + denoiser on CPU (recommended)
denoiser = SherpaOnnxDenoiserProvider(
    SherpaOnnxDenoiserConfig(model="gtcrn_simple.onnx", provider="cpu")
)
vad = SherpaOnnxVADProvider(
    SherpaOnnxVADConfig(model="ten-vad.onnx", provider="cpu")
)
stt = SherpaOnnxSTTProvider(
    SherpaOnnxSTTConfig(
        tokens="tokens.txt",
        encoder="encoder.onnx",
        decoder="decoder.onnx",
        joiner="joiner.onnx",
        provider="cuda",
    )
)
tts = SherpaOnnxTTSProvider(
    SherpaOnnxTTSConfig(model="vits.onnx", tokens="tokens.txt", provider="cuda")
)

pipeline_config = AudioPipelineConfig(denoiser=denoiser, vad=vad)
```

If CUDA is unavailable at runtime, ONNX Runtime falls back to CPU automatically.

---

## SherpaOnnxDenoiserProvider

Neural speech enhancement with GTCRN (the default) or DPDFNet. Removes background noise from microphone audio before it reaches VAD and STT.

Unlike `RNNoiseDenoiserProvider` (which requires `librnnoise` installed via apt/brew), the sherpa-onnx denoiser is a pure-Python dependency.

### Quick start

```python
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.denoiser.sherpa_onnx import (
    SherpaOnnxDenoiserConfig,
    SherpaOnnxDenoiserProvider,
)

denoiser = SherpaOnnxDenoiserProvider(
    SherpaOnnxDenoiserConfig(model="path/to/gtcrn_simple.onnx")
)

config = AudioPipelineConfig(denoiser=denoiser)
```

### DPDFNet

`model_type="dpdfnet"` loads one of the DPDFNet exports sherpa-onnx publishes in its
`speech-enhancement-models` release: `dpdfnet_baseline`, `dpdfnet2`, `dpdfnet4`, `dpdfnet8`
(16 kHz) or `dpdfnet2_48khz_hr` (48 kHz). It needs sherpa-onnx 1.12.30 or later.

```python
denoiser = SherpaOnnxDenoiserProvider(
    SherpaOnnxDenoiserConfig(model_type="dpdfnet", model="path/to/dpdfnet2.onnx")
)
```

- It streams, keeping its state per audio stream, and its output trails the input by a
  constant 50 ms (40 ms inside the model, 10 ms of overlap-add).
- It denoises only at the model's own rate: set the pipeline's resampler to it (16 kHz,
  or 48 kHz for `dpdfnet2_48khz_hr`). A frame at another rate passes through unchanged,
  with a warning.
- The silence gate stops the model once the input has stayed quiet for 100 ms, so its
  delayed output is not cut off.
- `context_frames` is GTCRN-only: a DPDFNet config refuses another value.

Measured on one core of an i7-13700KF, 20 ms frames, SI-SDR on four 5 s clips mixed at
5 dB SNR (the noisy input scores 5.0 dB):

| Model | Recorded noise | White noise | ms per frame (RTF) |
|---|---|---|---|
| GTCRN (sliding window) | 7.3 dB | 9.4 dB | 1.96 (0.098) |
| `dpdfnet2` | 10.8 dB | 11.5 dB | 2.51 (0.125) |
| `dpdfnet_baseline` | 10.5 dB | 11.4 dB | 1.47 (0.073) |

### Download the models

```bash
BASE=https://github.com/k2-fsa/sherpa-onnx/releases/download/speech-enhancement-models
wget $BASE/gtcrn_simple.onnx
wget $BASE/dpdfnet2.onnx            # or dpdfnet_baseline / dpdfnet4 / dpdfnet8 / dpdfnet2_48khz_hr
```

GitHub publishes the DPDFNet files' sha256 on the release page; the release's
`checksum.txt` lists `gtcrn_simple.onnx`'s.

### Configuration

All parameters are set via `SherpaOnnxDenoiserConfig`:

| Parameter | Default | Description |
|---|---|---|
| `model_type` | `"gtcrn"` | `"gtcrn"` or `"dpdfnet"`; an unknown type is refused. |
| `model` | `""` | Path to the model file: `gtcrn_simple.onnx`, or a DPDFNet export (required for DPDFNet). |
| `num_threads` | `1` | CPU threads for inference. |
| `provider` | `"cpu"` | ONNX execution provider (`"cpu"` or `"cuda"`). |
| `context_frames` | `3` | GTCRN only: preceding frames given as temporal context (see below). |
| `silence_threshold` | `0.005` | RMS (float32 scale, about −46 dBFS) below which inference is skipped; `0` disables. |

### How it works

GTCRN is an offline model — it needs temporal context to distinguish speech from noise. Processing isolated 20ms frames produces poor quality and boundary artifacts. The provider uses a **sliding context window** to give the model enough context:

1. Each audio frame is converted from int16 PCM to float32 samples in [-1, 1].
2. The samples are appended to a sliding context buffer (up to `context_frames` preceding frames).
3. `OfflineSpeechDenoiser.run(context_buffer, sample_rate)` processes the full context window.
4. Only the **last frame's portion** of the denoised output is extracted — no latency is added.
5. The denoised float32 samples are converted back to int16 PCM.
6. A new `AudioFrame` is returned with the cleaned audio, preserving all metadata.
7. On any error, the original frame is returned unchanged (pass-through).

The default `context_frames=3` (60 ms at 16 kHz) gives the model's recurrent layers enough warmup context while keeping inference about three times faster than the former 200 ms window. DPDFNet needs none of this: it streams.

### Lazy initialization

The denoiser is created lazily on the first call to `process()`, not at construction time. This means:

- Constructing `SherpaOnnxDenoiserProvider` is fast.
- Model loading happens only when audio actually arrives.
- `sherpa-onnx` must be installed at construction time (the import check happens in `__init__`).

### Lazy loader

```python
from roomkit.voice import get_sherpa_onnx_denoiser_provider, get_sherpa_onnx_denoiser_config

SherpaOnnxDenoiserProvider = get_sherpa_onnx_denoiser_provider()
SherpaOnnxDenoiserConfig = get_sherpa_onnx_denoiser_config()

denoiser = SherpaOnnxDenoiserProvider(
    SherpaOnnxDenoiserConfig(model="gtcrn_simple.onnx")
)
```

### Denoiser + VAD together

```python
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.denoiser.sherpa_onnx import (
    SherpaOnnxDenoiserConfig,
    SherpaOnnxDenoiserProvider,
)
from roomkit.voice.pipeline.vad.sherpa_onnx import (
    SherpaOnnxVADConfig,
    SherpaOnnxVADProvider,
)

denoiser = SherpaOnnxDenoiserProvider(
    SherpaOnnxDenoiserConfig(model="gtcrn_simple.onnx")
)
vad = SherpaOnnxVADProvider(
    SherpaOnnxVADConfig(model="ten-vad.onnx")
)

config = AudioPipelineConfig(denoiser=denoiser, vad=vad)
```

---

## SherpaOnnxVADProvider

Neural voice activity detection using [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx). Supports **TEN-VAD** and **Silero VAD** models for production-quality speech detection that works reliably in noisy environments.

Unlike `EnergyVADProvider` (simple RMS thresholding), the neural VAD can distinguish speech from loud non-speech noise like keyboard typing, music, or HVAC.

### Quick start

```python
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.vad.sherpa_onnx import SherpaOnnxVADConfig, SherpaOnnxVADProvider

vad = SherpaOnnxVADProvider(
    SherpaOnnxVADConfig(
        model="path/to/ten-vad.onnx",
        model_type="ten",
    )
)

config = AudioPipelineConfig(vad=vad)
```

### Download a model

```bash
# TEN-VAD (recommended — fast, low latency)
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/ten-vad.onnx

# Silero VAD
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/silero_vad.onnx
```

### Configuration

All parameters are set via `SherpaOnnxVADConfig`:

| Parameter | Default | Description |
|---|---|---|
| `model` | `""` | Path to the `.onnx` model file. |
| `model_type` | `"ten"` | Model architecture: `"ten"` or `"silero"`. |
| `threshold` | `0.35` | Speech probability threshold (0–1). Lower = more sensitive. Suits denoised audio; raise to `0.5` without a denoiser. |
| `silence_threshold_ms` | `500` | Consecutive silence in ms to trigger `SPEECH_END`. |
| `min_speech_duration_ms` | `250` | Minimum speech duration to emit. Shorter segments are discarded. |
| `speech_pad_ms` | `1000` | Pre-roll buffer — audio from before detection included in the segment. TEN-VAD detects speech 0.4–0.9 s after it starts, so a shorter pre-roll cuts the first word. |
| `max_speech_duration` | `20.0` | Maximum speech segment length in seconds (sherpa-internal). |
| `energy_silence_rms` | `20.0` | RMS below which audio counts as silence: `silence_threshold_ms` of it forces `SPEECH_END` even when the model still reports speech. `0` disables it. |
| `sherpa_min_silence_duration` | `0.05` | sherpa-onnx internal hysteresis, in seconds. Kept low; `silence_threshold_ms` does the debounce. |
| `sherpa_min_speech_duration` | `0.1` | sherpa-onnx internal minimum speech duration, in seconds. |
| `sample_rate` | `16000` | Expected audio sample rate. |
| `num_threads` | `1` | CPU threads for inference. |
| `provider` | `"cpu"` | ONNX execution provider (`"cpu"` or `"cuda"`). |

### How it works

The provider uses sherpa-onnx's `VoiceActivityDetector` with frame-level `is_speech_detected()` plus its own state machine:

1. Each audio frame is converted from int16 PCM to float32 and fed to the detector.
2. `is_speech_detected()` returns whether the current frame contains speech.
3. The state machine tracks idle/speaking transitions:
   - **Idle → Speaking**: emits `SPEECH_START` immediately (no delay).
   - **Speaking → Idle**: after `silence_threshold_ms` of consecutive non-speech, emits `SPEECH_END` with accumulated `audio_bytes`.
4. A pre-roll buffer keeps recent frames so speech onset isn't clipped.
5. Segments shorter than `min_speech_duration_ms` are silently discarded.

This gives instant `SPEECH_START` events (not delayed until a segment completes) and full control over timing parameters.

### TEN-VAD vs Silero VAD

| | TEN-VAD | Silero VAD |
|---|---|---|
| Latency | Very low | Low |
| Model size | ~2 MB | ~2 MB |
| Accuracy | Excellent | Excellent |
| Use case | Real-time voice assistants | General purpose |

Both models work well. TEN-VAD is recommended for real-time applications where latency matters.

### Lazy initialization

The sherpa-onnx detector is created lazily on the first call to `process()`, not at construction time. This means:

- Importing and constructing `SherpaOnnxVADProvider` is fast.
- Model loading happens only when audio actually arrives.
- `sherpa-onnx` must be installed at construction time (the import check happens in `__init__`).

### Fallback pattern

Use neural VAD when available, fall back to energy-based:

```python
import os

from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.vad.energy import EnergyVADProvider

vad_model = os.environ.get("VAD_MODEL", "")
if vad_model:
    from roomkit.voice.pipeline.vad.sherpa_onnx import (
        SherpaOnnxVADConfig,
        SherpaOnnxVADProvider,
    )

    vad = SherpaOnnxVADProvider(
        SherpaOnnxVADConfig(
            model=vad_model,
            model_type=os.environ.get("VAD_MODEL_TYPE", "ten"),
            threshold=float(os.environ.get("VAD_THRESHOLD", "0.5")),
        )
    )
else:
    vad = EnergyVADProvider(energy_threshold=300.0)

config = AudioPipelineConfig(vad=vad)
```

### Lazy loader

If you want to avoid importing sherpa-onnx at module level:

```python
from roomkit.voice import get_sherpa_onnx_vad_provider, get_sherpa_onnx_vad_config

SherpaOnnxVADProvider = get_sherpa_onnx_vad_provider()
SherpaOnnxVADConfig = get_sherpa_onnx_vad_config()

vad = SherpaOnnxVADProvider(SherpaOnnxVADConfig(model="ten-vad.onnx"))
```

---

## SherpaOnnxSTTProvider

Speech-to-text using [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx). Supports **transducer** models (streaming + batch) and **Whisper** models (batch only). All inference runs locally — no cloud APIs.

### Quick start

```python
from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider

stt = SherpaOnnxSTTProvider(
    SherpaOnnxSTTConfig(
        mode="transducer",
        encoder="path/to/encoder.onnx",
        decoder="path/to/decoder.onnx",
        joiner="path/to/joiner.onnx",
        tokens="path/to/tokens.txt",
    )
)
```

### Configuration

| Parameter | Default | Description |
|---|---|---|
| `mode` | `"transducer"` | Recognition mode: `"transducer"` or `"whisper"`. |
| `tokens` | `""` | Path to `tokens.txt`. |
| `encoder` | `""` | Path to encoder `.onnx` model. |
| `decoder` | `""` | Path to decoder `.onnx` model. |
| `joiner` | `""` | Path to joiner `.onnx` model (transducer only). |
| `model_type` | `""` | Model type hint (e.g. `"nemo_transducer"` for NeMo models). When set, the model is offline-only. |
| `language` | `"en"` | Language code (Whisper only). |
| `sample_rate` | `16000` | Expected audio sample rate. |
| `num_threads` | `2` | CPU threads for inference. |
| `provider` | `"cpu"` | ONNX execution provider (`"cpu"` or `"cuda"`). |
| `enable_endpoint_detection` | `True` | Enable endpoint detection (streaming transducer only). |
| `rule1_min_trailing_silence` | `2.4` | Endpoint rule 1 — trailing silence threshold (seconds). |
| `rule2_min_trailing_silence` | `1.2` | Endpoint rule 2 — trailing silence with text (seconds). |
| `rule3_min_utterance_length` | `20.0` | Endpoint rule 3 — minimum utterance length (seconds). |
| `tail_padding_s` | `0.66` | Silence fed to a streaming transducer before its input ends. The model decodes its last frames only with audio behind them; without it the last words are cut ("Hello" → "Hell"). |

### Streaming vs offline

| Mode | Streaming | Batch `transcribe()` | Use case |
|---|---|---|---|
| `transducer` (no `model_type`) | Yes | Yes | Real-time partial results (Zipformer, etc.) |
| `transducer` + `model_type` | No | Yes | High-accuracy offline models (NeMo TDT, Parakeet) |
| `whisper` | No | Yes | Whisper-family models |

When streaming is not supported, VoiceChannel uses VAD to segment speech and calls `transcribe()` on each complete utterance.

---

## SherpaOnnxTTSProvider

Text-to-speech using [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx), on the CPU or a CUDA GPU. Two model families: **VITS/Piper** (the default) and **Kokoro** (`model_type="kokoro"`). Audio streams sentence by sentence, at the model's own sample rate.

### Piper (VITS)

```python
from roomkit.voice.tts.sherpa_onnx import SherpaOnnxTTSConfig, SherpaOnnxTTSProvider

tts = SherpaOnnxTTSProvider(
    SherpaOnnxTTSConfig(
        model="fr_FR-siwis-medium.onnx",
        tokens="tokens.txt",
        data_dir="espeak-ng-data",
    )
)
```

### Kokoro

[Kokoro-82M](https://huggingface.co/hexgrad/Kokoro-82M) (Apache 2.0) speaks 24 kHz audio with 54 voices in v1.0: American and British English, Spanish, French, Hindi, Italian, Japanese, Brazilian Portuguese and Mandarin. sherpa-onnx packages it with its voices, lexicons and espeak-ng data:

```bash
# int8, 126 MB (the float model is kokoro-multi-lang-v1_0, 333 MB)
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/kokoro-int8-multi-lang-v1_0.tar.bz2
tar xf kokoro-int8-multi-lang-v1_0.tar.bz2
```

```python
d = "kokoro-int8-multi-lang-v1_0"
tts = SherpaOnnxTTSProvider(
    SherpaOnnxTTSConfig(
        model_type="kokoro",
        model=f"{d}/model.int8.onnx",
        voices=f"{d}/voices.bin",
        tokens=f"{d}/tokens.txt",
        data_dir=f"{d}/espeak-ng-data",
        lexicon=f"{d}/lexicon-us-en.txt,{d}/lexicon-zh.txt",
        lang="fr",
        speaker_id=30,  # ff_siwis
    )
)
```

- **`lang` is required for a language other than English or Chinese.** Without it, the multi-lang model reads any other text as English: French comes out mispronounced. It is an espeak-ng language name (`"fr"`, `"es"`, `"it"`, `"pt-BR"`, `"hi"`); French and English are the ones checked against a transcription.
- **A voice is an index** into `voices.bin` (`speaker_id`, or `voice="30"` per call). sherpa-onnx prints the table when the model loads with `debug`; in v1.0, `af_heart` is 3, `ef_dora` 28, `ff_siwis` 30, `if_sara` 35, `pf_dora` 42. A voice keeps its accent: pick one of the text's language.
- On two threads of a desktop CPU the int8 model renders a sentence in about two thirds of its spoken length.

### Configuration

| Parameter | Default | Description |
|---|---|---|
| `model_type` | `"vits"` | `"vits"` (VITS/Piper) or `"kokoro"`. |
| `model` | `""` | Path to the `.onnx` model. |
| `tokens` | `""` | Path to `tokens.txt`. |
| `data_dir` | `""` | Path to the espeak-ng data directory (Piper and Kokoro). |
| `lexicon` | `""` | Optional lexicon; several, comma-separated, for a multi-lang Kokoro. |
| `voices` | `""` | Kokoro's `voices.bin` (required with `"kokoro"`). |
| `lang` | `""` | Kokoro's espeak-ng language for text that is neither English nor Chinese. |
| `speaker_id` | `0` | Speaker of a multi-speaker model; a voice index for Kokoro. |
| `speed` | `1.0` | Speech speed multiplier. |
| `sample_rate` | `22050` | The rate a caller sets its output up for (Kokoro: 24000). Audio always carries the model's own rate. |
| `num_threads` | `2` | CPU threads for inference. |
| `provider` | `"cpu"` | ONNX execution provider (`"cpu"` or `"cuda"`). |

---

## NVIDIA Parakeet TDT

[NVIDIA Parakeet TDT 0.6B v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3) is a 600M-parameter FastConformer-TDT model with excellent accuracy (6.34% average WER). It supports **25 European languages** with automatic language detection, punctuation, and capitalization.

> **Note:** Parakeet TDT is an **offline model** — it does not support streaming. In the voice pipeline, VAD segments speech into utterances, then each utterance is transcribed in a single batch call. Latency depends on utterance length and hardware.

### Download the model

A pre-converted int8-quantized version is available (~640 MB):

```bash
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/asr-models/sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8.tar.bz2
tar xf sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8.tar.bz2
```

This contains:

| File | Size | Description |
|---|---|---|
| `encoder.int8.onnx` | 622 MB | FastConformer encoder |
| `decoder.int8.onnx` | 12 MB | TDT decoder |
| `joiner.int8.onnx` | 6.1 MB | Joiner network |
| `tokens.txt` | 92 KB | SentencePiece vocabulary (8192 tokens) |

### Quick start

```python
from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider

stt = SherpaOnnxSTTProvider(
    SherpaOnnxSTTConfig(
        mode="transducer",
        model_type="nemo_transducer",
        encoder="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/encoder.int8.onnx",
        decoder="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/decoder.int8.onnx",
        joiner="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/joiner.int8.onnx",
        tokens="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/tokens.txt",
    )
)
```

The `model_type="nemo_transducer"` tells sherpa-onnx how to handle the NeMo model format and disables streaming (since TDT requires full-context attention).

### Full pipeline example

```python
from roomkit import VoiceChannel
from roomkit.voice.backends.local import LocalAudioBackend
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.vad.sherpa_onnx import SherpaOnnxVADConfig, SherpaOnnxVADProvider
from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider

vad = SherpaOnnxVADProvider(
    SherpaOnnxVADConfig(model="ten-vad.onnx")
)
stt = SherpaOnnxSTTProvider(
    SherpaOnnxSTTConfig(
        mode="transducer",
        model_type="nemo_transducer",
        encoder="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/encoder.int8.onnx",
        decoder="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/decoder.int8.onnx",
        joiner="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/joiner.int8.onnx",
        tokens="sherpa-onnx-nemo-parakeet-tdt-0.6b-v3-int8/tokens.txt",
        provider="cuda",  # recommended for 600M model
    )
)
backend = LocalAudioBackend(
    input_sample_rate=16000,
    output_sample_rate=22050,
)

voice = VoiceChannel(
    "voice",
    stt=stt,
    tts=...,  # your TTS provider
    backend=backend,
    pipeline=AudioPipelineConfig(vad=vad),
)
```

### Supported languages

Bulgarian, Croatian, Czech, Danish, Dutch, English, Estonian, Finnish, French, German, Greek, Hungarian, Italian, Latvian, Lithuanian, Maltese, Polish, Portuguese, Romanian, Russian, Slovak, Slovenian, Spanish, Swedish, Ukrainian.

Language is detected automatically — no configuration needed.

### Performance tips

- **Use CUDA** for STT inference — the 600M model benefits significantly from GPU acceleration.
- **Use int8** (the default download) for ~2x faster inference with minimal quality loss.
- **Warmup** before the first interaction to avoid cold-start latency:

```python
await stt.warmup()  # loads the model into memory
```

---

## Examples

- [`examples/kokoro_tts_room.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/kokoro_tts_room.py) — a room that speaks every line typed into it with Kokoro, in the language and voice you pick.
- [`examples/voice_parakeet_tdt.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_parakeet_tdt.py) — local voice assistant with NVIDIA Parakeet TDT 0.6B v3 STT + sherpa-onnx TTS/VAD + Ollama/vLLM.
- [`examples/voice_local_onnx_vllm.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_local_onnx_vllm.py) — fully local voice assistant with sherpa-onnx STT/TTS/VAD + Ollama/vLLM, CUDA support. Supports optional [smart-turn detection](smart-turn.md) via `SMART_TURN_MODEL` env var.
- [`examples/voice_smart_turn.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_smart_turn.py) — audio-native turn detection with smart-turn + sherpa-onnx STT/TTS/VAD. See the [Smart Turn Detection guide](smart-turn.md).
- [`examples/voice_sherpa_onnx_vad.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_sherpa_onnx_vad.py) — standalone local mic demo with neural VAD + optional denoiser.
- [`examples/voice_cloud.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_cloud.py) — cloud voice assistant with Deepgram STT + ElevenLabs TTS + Claude (set `VAD_MODEL` and/or `DENOISE_MODEL` env vars).
