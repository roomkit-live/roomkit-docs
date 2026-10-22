# Local Models

RoomKit knows the files of the local models its providers run: sherpa-onnx
speech-to-text, text-to-speech, VAD and denoiser, and Smart Turn. Each model is
pinned to its URLs and their SHA-256. Download the ones you use once, while
online; from then on a provider config is built by the model's name and needs
no network.

```bash
python -m roomkit.local_models list
python -m roomkit.local_models download zipformer-en-2023-06-26 kokoro-int8-multi-lang-v1_0 ten-vad smart-turn-v3.2-cpu
```

```python
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.turn import SmartTurnConfig, SmartTurnDetector
from roomkit.voice.pipeline.vad.sherpa_onnx import SherpaOnnxVADConfig, SherpaOnnxVADProvider
from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider
from roomkit.voice.tts.sherpa_onnx import SherpaOnnxTTSConfig, SherpaOnnxTTSProvider

stt = SherpaOnnxSTTProvider(SherpaOnnxSTTConfig.from_local_model("zipformer-en-2023-06-26"))
tts = SherpaOnnxTTSProvider(
    SherpaOnnxTTSConfig.from_local_model("kokoro-int8-multi-lang-v1_0", voice="af_heart")
)
pipeline = AudioPipelineConfig(
    vad=SherpaOnnxVADProvider(SherpaOnnxVADConfig.from_local_model("ten-vad")),
    turn_detector=SmartTurnDetector(SmartTurnConfig.from_local_model("smart-turn-v3.2-cpu")),
)
```

The complete agent, with a local LLM through llama.cpp and RNNoise, is
[`examples/voice_local_offline.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_local_offline.py).

## The models

| Name | Config | Download | What it is |
|---|---|---|---|
| `zipformer-en-2023-06-26` | `SherpaOnnxSTTConfig` | 73 MB | Streaming STT, English, zipformer transducer (int8) |
| `zipformer-en-20M-2023-02-17` | `SherpaOnnxSTTConfig` | 44 MB | Streaming STT, English, small (20M params); drops the first word of audio resampled from 48 kHz |
| `zipformer-en-kroko-2025-08-06` | `SherpaOnnxSTTConfig` | 57 MB | Streaming STT, English, Kroko |
| `zipformer-fr-kroko-2025-08-06` | `SherpaOnnxSTTConfig` | 57 MB | Streaming STT, French, Kroko (1.5 s tail padding) |
| `nemotron-3.5-asr-streaming-0.6b-560ms-int8` | `SherpaOnnxSTTConfig` | 475 MB | Streaming STT, multilingual, Nemotron 3.5; detects the language, `stream_language="fr"` fixes it |
| `parakeet-tdt-0.6b-v3-int8` | `SherpaOnnxSTTConfig` | 487 MB | Offline STT, 25 European languages, Parakeet TDT v3 |
| `whisper-tiny.en-int8` | `SherpaOnnxSTTConfig` | 104 MB | Offline STT, English, Whisper tiny.en |
| `whisper-base-int8` | `SherpaOnnxSTTConfig` | 161 MB | Offline STT, multilingual, Whisper base; detects the language, `language="fr"` fixes it |
| `moonshine-tiny-en-2026-02-27` | `SherpaOnnxSTTConfig` | 30 MB | Offline STT, English, Moonshine v2 tiny; about 0.05 s per sentence on a CPU |
| `moonshine-base-en-2026-02-27` | `SherpaOnnxSTTConfig` | 111 MB | Offline STT, English, Moonshine v2 base |
| `kokoro-int8-multi-lang-v1_0` | `SherpaOnnxTTSConfig` | 132 MB | Kokoro v1.0, 54 named voices, 24 kHz |
| `vits-piper-en_US-amy-low` | `SherpaOnnxTTSConfig` | 67 MB | Piper "amy", American English, 16 kHz |
| `vits-piper-fr_FR-siwis-medium` | `SherpaOnnxTTSConfig` | 67 MB | Piper "siwis", French, 22 kHz |
| `kitten-nano-en-v0_8-int8` | `SherpaOnnxTTSConfig` | 31 MB | Kitten nano v0.8, English, 8 named voices, 24 kHz |
| `kitten-mini-en-v0_8` | `SherpaOnnxTTSConfig` | 68 MB | Kitten mini v0.8, English, 8 named voices, 24 kHz |
| `ten-vad` | `SherpaOnnxVADConfig` | < 1 MB | TEN-VAD |
| `silero-vad` | `SherpaOnnxVADConfig` | < 1 MB | Silero VAD |
| `gtcrn-simple` | `SherpaOnnxDenoiserConfig` | < 1 MB | GTCRN speech enhancement |
| `dpdfnet2` | `SherpaOnnxDenoiserConfig` | 11 MB | DPDFNet2, streaming, for a 16 kHz pipeline |
| `dpdfnet2-48khz-hr` | `SherpaOnnxDenoiserConfig` | 11 MB | DPDFNet2, streaming, for a 48 kHz pipeline |
| `smart-turn-v3.2-cpu` | `SmartTurnConfig` | 9 MB | Smart Turn v3.2, CPU build (int8) |
| `smart-turn-v3.2-gpu` | `SmartTurnConfig` | 32 MB | Smart Turn v3.2, fp32 build for `provider="cuda"` |

The offline STT models (Parakeet, Whisper, Moonshine) transcribe each utterance once it
ends, behind a VAD; the streaming ones write partials as the person speaks.
Each entry was run once before it went in: the French Piper voice, for
instance, is transcribed word for word by Parakeet and Kroko.

`python -m roomkit.local_models list` prints the same table and marks the
models already downloaded; `--kind sherpa-onnx-tts` narrows it to one config.
From Python, `list_local_models()` returns them as `LocalModel`s (name,
kind, description, size, files, voices), and `is_downloaded()` says which are
there:

```python
from roomkit.local_models import SHERPA_ONNX_TTS, is_downloaded, list_local_models

for model in list_local_models(SHERPA_ONNX_TTS):
    print(model.name, model.size_mb, is_downloaded(model), model.voices[:3])
```

## Downloading

`download` fetches each file, checks it against its pinned SHA-256 and extracts
the archives, in a staging directory renamed into place only when complete: an
interrupted download leaves nothing half-written, and a file that does not
match is refused. A model already there is skipped, so the command is safe to
re-run. When RoomKit pins newer files for a model, the old copy reads as not
downloaded and the next `download` replaces it.

Models live in `$ROOMKIT_MODELS_DIR` when it is set, else in
`~/.cache/roomkit/models` (`$XDG_CACHE_HOME/roomkit/models`). Every command and
`from_local_model()` take `--models-dir` / `models_dir=` to use another one.

In a container, download at build time and the image runs offline:

```dockerfile
ENV ROOMKIT_MODELS_DIR=/models
RUN python -m roomkit.local_models download zipformer-en-2023-06-26 kokoro-int8-multi-lang-v1_0 ten-vad
```

`python -m roomkit.local_models path <name>` prints a downloaded model's
directory, for a script that needs the files themselves.

From Python, `download_local_model(name)` does the same as the command. It
blocks while it downloads: call it at setup, not from an event loop.

## Building a config by name

`from_local_model(name, ...)` exists on `SherpaOnnxSTTConfig`,
`SherpaOnnxTTSConfig`, `SherpaOnnxVADConfig`, `SherpaOnnxDenoiserConfig` and
`SmartTurnConfig`. It fills in the file paths and the settings the model needs
(its mode, model type, sample rate), and any other field is passed as a keyword
and wins over the model's settings:

```python
SherpaOnnxSTTConfig.from_local_model("zipformer-en-2023-06-26", provider="cuda", num_threads=4)
```

It never touches the network. A model that is not downloaded raises
`ModelNotDownloadedError`, whose message is the command that downloads it:

```
Local model 'ten-vad' is not downloaded in /home/me/.cache/roomkit/models.
Download it once, while online: python -m roomkit.local_models download ten-vad
```

A name that does not exist, or a model for another config (`ten-vad` given to
`SherpaOnnxSTTConfig`), raises `ValueError` listing the names that fit.

## Voices by name

A Kokoro voice is named rather than numbered. `voice=` sets the default voice,
and a voice name also works per synthesis, a `VoiceChannel(voice_map=...)`
entry included:

```python
config = SherpaOnnxTTSConfig.from_local_model(
    "kokoro-int8-multi-lang-v1_0", voice="ff_siwis", lang="fr"
)
config.speaker_id                     # 30
await tts.synthesize("Bonjour", voice="ff_siwis")
```

The names come from the model's own speaker table and fill
`SherpaOnnxTTSConfig.voice_names`; a speaker id given as a number still works.
`KOKORO_V1_VOICES` in `roomkit.local_models` lists the 54 Kokoro names in id
order, `KITTEN_V08_VOICES` the 8 Kitten ones (`expr-voice-2-f`...). A
voice keeps its accent: pick one of the text's language, and set `lang` for a
language other than English or Chinese.

## RNNoise without a system library

`RNNoiseDenoiserProvider` loads `librnnoise`. When the system has none, it uses
the copy the `pyrnnoise` wheel ships for Linux (x86_64, aarch64), macOS and
Windows:

```bash
pip install roomkit[rnnoise]
```

Only the native library is loaded; the `pyrnnoise` package is never imported.
Its declared dependencies (audiolab, matplotlib) are installed all the same. A
system copy (`apt install librnnoise0`, or a source build) is preferred when
present. Each pyrnnoise release ships a macOS wheel for the newest macOS only;
on macOS 15, pip installs 0.4.3, the last release built for it.

## What is not here

- **LLMs.** `LlamaCppAIProvider` downloads its GGUF model itself, through
  `llama-server`, on first start; pass the path of a `.gguf` file to run one
  you already hold.
- **Models without an entry.** Any sherpa-onnx or Smart Turn model still works
  through the config's path fields; the list holds the models RoomKit has
  checked end to end.
