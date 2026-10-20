# Phonon TTS Provider

[Phonon](https://huggingface.co/collections/gradium/phonon) is Gradium's family
of small text-to-speech models (31M and 40M parameters), built on Pocket TTS
and Kyutai's Mimi codec. It streams 24 kHz speech on the CPU and speaks
**English, French, German, Spanish and Portuguese**. Its Rust runtime,
[`ptts`](https://pypi.org/project/ptts/), needs neither torch nor a GPU: the
wheel depends on numpy only. `PhononTTSProvider` runs it in your process.

```bash
pip install roomkit[phonon]
```

The Linux and Windows x86_64 wheels need a CPU with AVX2 (x86-64-v3).

---

## Quick start

```python
from roomkit.voice.tts.phonon import PhononTTSConfig, PhononTTSProvider

tts = PhononTTSProvider(PhononTTSConfig(
    model="hf://gradium/phonon-40m-fr",
    language="fr",
    voice="Noémie",                  # from the voice catalog
))

await tts.warmup()                   # loads the model and fetches the voice

async for chunk in tts.synthesize_stream("Bonjour, comment puis-je vous aider ?"):
    ...                              # 16-bit PCM, 24 kHz, then an is_final marker

audio = await tts.synthesize("Au revoir.")   # a WAV data URL
```

`PhononTTSProvider()` with no config loads the runtime's default model,
Phonon 31m English.

---

## Models

| Model | Languages |
|-------|-----------|
| `hf://gradium/phonon-31m-en` (the default) | English |
| `hf://gradium/phonon-40m-en`, `-fr`, `-de`, `-es`, `-pt` | One each |
| `hf://gradium/phonon-40m-multilingual` | All five |

Each model page on Hugging Face states its licence. The 40m models are gated:
accept their terms on the model page, then run `hf auth login` or set
`HF_TOKEN`. Weights download on first use and are cached; a local checkpoint
directory works too, and downloads nothing.

---

## Configuration

| Field | Default | Meaning |
|-------|---------|---------|
| `model` | `None` | `hf://owner/name[@revision]` or a local checkpoint directory; `None` loads Phonon 31m English, pinned to one revision |
| `language` | `"en"` | How numbers, dates and symbols are spelled out: `en`, `fr`, `de`, `es`, `pt`, or `none` to speak the text as written. Match the model's language |
| `voice` | `None` | Default voice: a catalog name (any case, accents optional), its voice id, or a key of `voices`; `None` uses the model's own voice |
| `voices` | `{}` | Voice files (`.safetensors`) by name, registered when the model loads |
| `quant` | `"q8"` | Weight format; `q8` is the fastest on a CPU, `None` loads full precision |
| `temperature` | `None` | Sampling temperature; `None` keeps the runtime's (0.3) |
| `duration_delta` | `None` | Speaking rate: negative is faster, positive slower (`-0.3`) |

### Voices

The catalog [gradium/phonon-voices](https://huggingface.co/gradium/phonon-voices)
holds about a hundred voices with samples, 15 of them French. A voice
downloads the first time it is used; `warmup()` fetches the configured
default, so the first reply does not wait for it. `list_voices()` returns the
configured voice files, then the catalog, and filters it by language or text:

```python
french = await tts.list_voices(language="fr")
```

The catalog gives a name, an id, a language and a description for each voice,
no gender, so a gender filter matches none of them.

Phonon does not clone from a clip in the process. A voice designed or cloned
in [Gradium Studio](https://studio.gradium.ai) downloads as a `.safetensors`
file that works with every Phonon model:

```python
PhononTTSConfig(model="hf://gradium/phonon-40m-fr", language="fr",
                voices={"brand": "./brand.safetensors"}, voice="brand")
```

An unknown voice raises `LookupError` when the stream starts.

### Speed

Measured on a desktop (24-core x86 CPU), q8 weights, streaming, the runtime on
two threads (`ptts.set_num_threads(2)`):

| Model | First chunk | Speed |
|-------|-------------|-------|
| `phonon-31m-en` | 40–80 ms | 11–12x real time |
| `phonon-40m-fr` | 40–110 ms | 9.5x real time |
| `phonon-40m-multilingual` (French text) | 50–60 ms | 9.3x real time |

The full-precision weights (`quant=None`) ran at 9.6x where q8 ran at 12x.
`ptts.set_num_threads` sizes the runtime's thread pool for the
whole process: call it before the first model loads.

---

## Barge-in and concurrency

When the Voice Channel cancels playback (a barge-in), it closes the stream and
the provider stops the runtime's workers: about 50–70 ms in our runs.

The runtime streams several replies at once from one loaded model, so the
provider does not serialise them: two replies streamed side by side took 1.5 s
where one took 1.0 s.

Phonon does not consume the conversation context (`context_level` is `NONE`):
each reply is synthesized on its own. It takes a whole text, not a text
stream; the Voice Channel hands it one sentence at a time.

---

## A local French assistant

`examples/voice_local_phonon.py` runs a French voice assistant with
everything on the machine:

```
Mic → [AEC] → TEN-VAD → Kroko French STT (sherpa-onnx) → local LLM → Phonon (40m-fr) → Speaker
```

```bash
uv run --extra local-audio --extra webrtc-aec --extra llamacpp \
    --extra sherpa-onnx --extra phonon \
    python examples/voice_local_phonon.py

PHONON_VOICE=Lucien PHONON_SPEED=-0.2 uv run ... python examples/voice_local_phonon.py
CONSOLE=1 uv run ... --extra console python examples/voice_local_phonon.py   # console dashboard
```

`PHONON_MODEL` picks another model (`hf://gradium/phonon-40m-multilingual`),
and the header of the example lists every variable.
