# Pocket TTS Provider

[Pocket TTS](https://github.com/kyutai-labs/pocket-tts) is Kyutai's 100M-parameter
text-to-speech model. It streams 24 kHz speech in 80 ms chunks, faster than real
time on two CPU cores, and speaks **English, French, German, Portuguese, Italian
and Spanish**. It clones a voice from a short clip. `PocketTTSProvider` runs it in
your process, on the CPU or on a CUDA GPU.

```bash
pip install roomkit[pocket-tts]
```

On Linux, PyPI serves the CUDA build of PyTorch (about 3 GB of wheels). For a
CPU-only install, take torch from the PyTorch CPU index first:

```bash
pip install torch --index-url https://download.pytorch.org/whl/cpu
pip install roomkit[pocket-tts]
```

---

## Quick start

```python
from roomkit.voice.tts.pocket import PocketTTSConfig, PocketTTSProvider

tts = PocketTTSProvider(PocketTTSConfig(
    language="french",
    voices={"estelle": "estelle"},   # a pre-made voice
    device="cpu",                    # or "cuda"
))

await tts.warmup()                   # loads the model and the voice states

async for chunk in tts.synthesize_stream("Bonjour, comment puis-je vous aider ?"):
    ...                              # 16-bit PCM, 24 kHz, then an is_final marker

audio = await tts.synthesize("Au revoir.")   # a WAV data URL
```

---

## Configuration

| Field | Default | Meaning |
|-------|---------|---------|
| `language` | `"english"` | Model to load: `english`, `french`, `german`, `portuguese`, `italian`, `spanish`, each also as a larger `_24l` variant (`french_24l`) |
| `voices` | `{"alba": "alba"}` | Named voices, the first is the default. A value is a pre-made voice name, a local clip to clone, an `hf://` path, or a `.safetensors` voice state |
| `device` | `"cpu"` | `"cpu"` or a CUDA device (`"cuda"`, `"cuda:1"`) |
| `quantize` | `False` | int8 dynamic quantization, CPU only |
| `temperature` | `None` | Sampling temperature; `None` keeps the model's default |

One model speaks one language: to serve two languages, create two providers.

### Voices

Pre-made voices are listed in the [Pocket TTS README](https://github.com/kyutai-labs/pocket-tts#the-generate-command):
`estelle` is the French one, `alba` the default English one. Each voice has
its own licence, see [kyutai/tts-voices](https://huggingface.co/kyutai/tts-voices).

To clone a voice, pass a clean speech clip; its recording quality is
reproduced, so clean it first. Encoding a clip takes a second or two at
`warmup()`. Export it once, with the model's language, to load it instantly
afterwards (a voice state belongs to the model it was exported with):

```bash
uvx pocket-tts export-voice --language french my_voice.wav my_voice.safetensors
```

```python
PocketTTSConfig(language="french", voices={"brand": "./my_voice.safetensors"})
```

### CPU or GPU

Measured on a desktop (24-core x86 CPU, RTX 4070), `french` model, streaming:

| Setup | First chunk | Speed |
|-------|-------------|-------|
| CPU, 2 threads | ~80 ms | 3.9x real time |
| CPU, 2 threads, `quantize=True` | ~50 ms | 7x real time |
| GPU (`device="cuda"`) | ~20 ms | 12.8x real time |
| `french_24l`, CPU, 2 threads | ~240 ms | 1.3x real time |
| `french_24l`, GPU | ~35 ms | 5.6x real time |

The standard model is enough for a conversation on CPU. The `_24l` models sound
better, but on CPU they barely keep up with playback. In our French tests,
`french_24l` also often stopped a long sentence at its first comma, on CPU
and GPU alike. Kyutai does not officially support GPUs, but the model is a
plain `torch` module and `device="cuda"` moves it there.

If a GPU run fails with `CUDNN_STATUS_SUBLIBRARY_VERSION_MISMATCH`, a system
cuDNN is shadowing the one bundled with PyTorch. Set
`torch.backends.cudnn.enabled = False` before loading the model.

---

## Barge-in and concurrency

The model is not thread-safe: the provider runs one generation at a time
behind a lock. When the Voice Channel cancels playback (a barge-in), it closes
the stream. The provider then tells the model to stop and waits for its
thread before the next reply starts. In our runs this took about 20 ms.

Pocket TTS does not consume the conversation context (`context_level` is
`NONE`): each reply is synthesized on its own. For a TTS that hears the
dialogue, see [Vui Nano](tts-context.md) (English only).

---

## A local French assistant

`examples/voice_local_pocket_fr.py` runs a French voice assistant with
everything on the machine:

```
Mic → [AEC] → TEN-VAD → Kroko French STT (sherpa-onnx) → Ollama → Pocket TTS (french) → Speaker
```

```bash
uv run --extra local-audio --extra webrtc-aec --extra ollama \
    --extra sherpa-onnx --extra pocket-tts \
    python examples/voice_local_pocket_fr.py

POCKET_DEVICE=cuda uv run ... python examples/voice_local_pocket_fr.py   # on the GPU
```

The Kroko French transducer drops the last word of an utterance unless it gets
about 1.5 s of tail silence, so the example sets `tail_padding_s=1.5` on
`SherpaOnnxSTTConfig`. The padding costs compute, not waiting time.
