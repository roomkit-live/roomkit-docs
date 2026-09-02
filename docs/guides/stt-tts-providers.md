# STT & TTS Providers

RoomKit's voice pipeline uses pluggable Speech-to-Text (STT) and Text-to-Speech (TTS) providers. This guide covers all built-in providers, their configuration, and streaming capabilities.

## STT Provider ABC

```python
from __future__ import annotations

from roomkit.voice.stt import STTProvider, TranscriptionResult


class STTProvider(ABC):
    @property
    def name(self) -> str: ...

    @property
    def supports_streaming(self) -> bool: ...

    @property
    def supports_language_override(self) -> bool: ...
        """Whether transcribe/transcribe_stream honour a per-call language."""

    async def transcribe(self, audio, *, language=None) -> TranscriptionResult:
        """Batch transcription — send all audio, get full text."""

    async def transcribe_stream(
        self, audio_stream, *, language=None
    ) -> AsyncIterator[TranscriptionResult]:
        """Streaming transcription — get partial results in real time."""

    async def warmup(self) -> None:
        """Pre-load models (optional)."""

    async def close(self) -> None:
        """Release resources."""
```

`language` on a call overrides the provider's configured language for that
call only. A provider that cannot honour it answers `False` to
`supports_language_override`, and `VoiceChannel` never passes a language to
such a provider — an implementation written against the older signature
keeps working unchanged.

### TranscriptionResult

```python
@dataclass
class TranscriptionResult:
    text: str
    is_final: bool = True
    confidence: float | None = None
    language: str | None = None
    words: list[dict[str, Any]] = []
    is_speech_start: bool = False
```

`language` is what the provider **reports** — what it detected when it was
asked to detect — never an echo of the language it was configured with. A
Deepgram stream pinned to `fr-CA` reports nothing; one opened in `multi`
reports the language most words carried.

---

## Deepgram (Cloud API)

The most feature-rich cloud STT provider. Real-time streaming with interim results, keyword boosting, and entity detection.

```python
from __future__ import annotations

from roomkit.voice.stt.deepgram import DeepgramConfig, DeepgramSTTProvider

stt = DeepgramSTTProvider(
    config=DeepgramConfig(
        api_key="your-api-key",
        model="nova-3",
        language="en",
        punctuate=True,
        smart_format=True,
        interim_results=True,
        endpointing=300,         # Silence duration (ms) before endpoint
        vad_events=True,         # Emit speech_start events
    )
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `model` | `"nova-2"` | Model name (`"nova-3"` for `multi` and keyterms) |
| `language` | `"en"` | Language code — `"multi"` detects and code-switches (Nova-3) |
| `punctuate` | `True` | Add punctuation |
| `smart_format` | `True` | Smart formatting (dates, numbers) |
| `numerals` | `False` | Convert numbers to digits |
| `interim_results` | `True` | Include partial results while speaking |
| `endpointing` | `300` | Silence ms before utterance end, or `False` to disable |
| `utterance_end_ms` | `None` | Additional utterance end signal |
| `vad_events` | `True` | Emit VAD events |
| `diarize` | `False` | Speaker diarization |
| `filler_words` | `False` | Include "um", "uh" |
| `keywords` | `[]` | Keywords to boost recognition |
| `keyterm` | `[]` | Key terms (Nova-3) |
| `profanity_filter` | `False` | Filter profanity |
| `redact` | `[]` | Redaction rules (e.g., `["pci", "ssn"]`) |
| `detect_entities` | `False` | Detect named entities |

**Streaming events**: `SpeechStarted`, `Results` (partial + final), `UtteranceEnd`

**Batch mode**: HTTP POST to `/listen` endpoint.

**Streaming mode**: WebSocket with real-time partials and finals.

### Language: detect, then lock

Nova-3 transcribes code-switched speech with `language="multi"`, and reports
what it heard — `TranscriptionResult.language` carries the language most words
were tagged with. A stream pinned to the speaker's language (`fr-CA`) is
measurably better than `multi`, but Deepgram fixes the language in the
WebSocket URL: it holds for the life of a stream. RoomKit opens one stream per
utterance (VAD mode) or per turn (continuous mode), so the language can change
between them — never inside one.

`VoiceChannel.set_stt_language` chooses the language for one session from its
next stream on; `None` returns to the provider's configuration. The typical
caller is an `ON_TRANSCRIPTION` hook reading `event.language`:

```python
stt = DeepgramSTTProvider(DeepgramConfig(api_key="...", model="nova-3", language="multi"))
voice = VoiceChannel("voice", stt=stt, backend=backend, pipeline=AudioPipelineConfig())

@kit.hook(HookTrigger.ON_TRANSCRIPTION)
async def pin_language(event, ctx):
    # A code while the stream detects, None once it is pinned
    if event.language == "fr":
        voice.set_stt_language(event.session, "fr-CA")
    return HookResult.allow()
```

`STTLanguageLock` packages that loop, with a way back:

```python
from roomkit import STTLanguageLock

voice = VoiceChannel(
    "voice",
    stt=stt,
    backend=backend,
    pipeline=AudioPipelineConfig(),
    stt_language_lock=STTLanguageLock(
        detect_language="multi",     # every session starts here
        prefer={"fr": "fr-CA"},      # reported -> locked
        lock_after=1,                # agreeing finals before locking
        release_after=2,             # consecutive misses before detecting again
        min_confidence=0.5,          # a final below this is a miss
    ),
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `detect_language` | `"multi"` | Language every session starts in, and returns to |
| `prefer` | `{}` | Reported code → locked code (Deepgram reports `fr`, you want `fr-CA`) |
| `lock_after` | `1` | Consecutive finals reporting the same language before locking |
| `release_after` | `2` | Consecutive misses before releasing back to `detect_language` |
| `min_confidence` | `0.5` | A final below this confidence is a miss |

A miss is a final with no text, a confidence below `min_confidence`, or a
reported language other than the lock; a fitting final resets the count. A
locked stream reports no language, so the way back is noise: a caller who
switches to English on a `fr-CA` stream produces empties and low-confidence
finals, and two of those in a row reopen the session in `multi`.

Where the change lands:

- **VAD mode** — the next utterance. A stream already open stays open, so the
  utterance in progress is not cut in two.
- **Continuous mode** — right away: the current cycle is ended and the loop
  reconnects with the new language. Audio arriving in the gap is kept.
- **Batch mode** — the next `flush_stt()`.

Runnable: `examples/voice_deepgram_language_lock.py`.

---

## SherpaOnnx (Local/Offline)

Run STT locally without API calls using ONNX models. Supports transducer (streaming) and Whisper (batch) modes.

```python
from __future__ import annotations

from roomkit.voice.stt.sherpa_onnx import SherpaOnnxSTTConfig, SherpaOnnxSTTProvider

# Streaming transducer mode
stt = SherpaOnnxSTTProvider(
    config=SherpaOnnxSTTConfig(
        mode="transducer",
        tokens="path/to/tokens.txt",
        encoder="path/to/encoder.onnx",
        decoder="path/to/decoder.onnx",
        joiner="path/to/joiner.onnx",
        sample_rate=16000,
        num_threads=2,
        provider="cpu",                        # or "cuda"
        enable_endpoint_detection=True,
        rule1_min_trailing_silence=2.4,        # Seconds
        rule2_min_trailing_silence=1.2,
        rule3_min_utterance_length=20.0,
    )
)

# Batch Whisper mode (no streaming)
stt_whisper = SherpaOnnxSTTProvider(
    config=SherpaOnnxSTTConfig(
        mode="whisper",
        tokens="path/to/tokens.txt",
        encoder="path/to/encoder.onnx",
        decoder="path/to/decoder.onnx",
        language="en",
        task="transcribe",                     # or "translate" for English translation
    )
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `mode` | required | `"transducer"` (streaming) or `"whisper"` (batch only) |
| `tokens` | required | Path to `tokens.txt` |
| `encoder` | required | Path to encoder ONNX model |
| `decoder` | required | Path to decoder ONNX model |
| `joiner` | `None` | Path to joiner ONNX (transducer only) |
| `sample_rate` | `16000` | Expected input sample rate |
| `num_threads` | `2` | CPU threads for inference |
| `provider` | `"cpu"` | ONNX runtime: `"cpu"` or `"cuda"` |
| `enable_endpoint_detection` | `True` | Detect utterance endpoints |
| `rule1_min_trailing_silence` | `2.4` | Silence threshold (seconds) for rule 1 |
| `rule2_min_trailing_silence` | `1.2` | Silence with text threshold |
| `rule3_min_utterance_length` | `20.0` | Min utterance length for rule 3 |

---

## Gradium (Cloud API)

Cloud STT with built-in server-side VAD and pre-connect buffering to avoid lost first words.

```python
from __future__ import annotations

from roomkit.voice.stt.gradium import GradiumSTTConfig, GradiumSTTProvider

stt = GradiumSTTProvider(
    config=GradiumSTTConfig(
        api_key="your-api-key",
        region="us",
        model_name="default",
        input_format="pcm",
        language="en",
        connect_buffer_ms=300,    # Buffer audio before WebSocket opens
        delay_in_frames=7,        # Processing delay (7-48, each = 80ms)
        vad_threshold=0.9,        # VAD inactivity threshold
        vad_steps=10,             # Steps above threshold to confirm end
        timeout_s=3.0,            # Server inactivity timeout
    )
)
```

**Streaming events**: `text` (partial), `end_text` (segment done), `step` (VAD heartbeat)

**Pre-connect buffering**: Accumulates real audio before opening the WebSocket, then sends a burst — avoids losing the first few words.

---

## Qwen3 ASR (Local/GPU)

HuggingFace-based ASR with optional vLLM backend for streaming.

```python
from __future__ import annotations

from roomkit.voice.stt.qwen3 import Qwen3ASRConfig, Qwen3ASRProvider

stt = Qwen3ASRProvider(
    config=Qwen3ASRConfig(
        model_id="Qwen/Qwen3-ASR-0.6B",
        backend="vllm",             # "vllm" for streaming, "transformers" for batch
        device_map="auto",
        dtype="bfloat16",
        language=None,               # None = auto-detect
        chunk_size_sec=2.0,          # Streaming chunk duration
        gpu_memory_utilization=0.3,
        max_new_tokens=2048,
    )
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `model_id` | `"Qwen/Qwen3-ASR-0.6B"` | HuggingFace model ID |
| `backend` | `"transformers"` | `"transformers"` (batch) or `"vllm"` (batch + streaming) |
| `dtype` | `"bfloat16"` | Model precision |
| `language` | `None` | Language code or `None` for auto-detect |
| `chunk_size_sec` | `2.0` | Streaming chunk duration (vLLM only) |
| `gpu_memory_utilization` | `0.3` | GPU memory fraction (vLLM only) |

---

## Gemini (Cloud API, batch only)

Gemini has no speech-to-text endpoint. Transcription is an *instruction* to a
multimodal model that accepts audio, so this provider is batch by construction:
it takes a complete recording and answers in one pass, in seconds. Google's own
audio documentation points at Cloud Speech-to-Text for dedicated real-time
transcription, and that stays the right advice for live turn-taking.

What the batch shape buys is what a streaming recogniser structurally cannot
give. The model sees the whole recording before it answers, so one pass returns
the transcript, the speaker turns and the timestamps together — no diarization
stage, no merge. This is the provider for meeting recordings, voicemail and
imported audio files.

```python
from __future__ import annotations

from roomkit.voice.stt.gemini import GeminiSTTConfig, GeminiSTTProvider

stt = GeminiSTTProvider(
    config=GeminiSTTConfig(
        api_key="your-gemini-api-key",
        model="gemini-3.6-flash",     # any multimodal model that accepts audio
        language="fr-CA",              # optional hint; detected otherwise
        diarize=True,                  # ask for speaker labels
        prompt="The product is spelled RoomKit.",   # optional vocabulary/format
    )
)

transcript = await stt.transcribe_recording("meeting.wav")
print(transcript.language)                    # "fr-CA"
for turn in transcript.segments:
    print(f"[{turn.start}-{turn.end}] {turn.speaker}: {turn.text}")
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `api_key` | *(required)* | Gemini API key (`GEMINI_API_KEY`) |
| `model` | `"gemini-3.6-flash"` | A multimodal model that accepts audio input |
| `language` | `None` | BCP-47 hint; unset, the model identifies and reports it |
| `diarize` | `True` | Ask for `Speaker 1`, `Speaker 2`, … labels |
| `prompt` | `None` | Extra instruction: vocabulary, formatting rules |
| `timeout` | `600.0` | Per-request timeout in seconds |
| `max_inline_bytes` | `15 MiB` | Above this, the recording is uploaded via the Files API |

### Two methods, two shapes

`transcribe()` is the `STTProvider` contract and returns flat text, dropping the
structure. `transcribe_recording()` returns the whole `Transcript` — the
detected language plus one `TranscriptSegment` per speaker turn, with `.text`
(labelled) and `.plain_text` (words only) helpers. It also accepts a file path,
which `transcribe()` does not.

### Input paths

Raw `AudioChunk`/`AudioFrame` audio is sent inline as PCM; a `data:` URL is sent
inline as-is; a local path is inlined below `max_inline_bytes` and uploaded
through the Files API above it (and deleted afterwards, rather than left to
expire). Arbitrary `http(s)` URLs are **refused**, not fetched: dereferencing a
caller-supplied URL would make the provider an SSRF vector. Upload the file or
pass the bytes.

### Limits worth knowing

`supports_streaming` is `False`, so a `VoiceChannel` transcribes on `SPEECH_END`
instead of streaming partials — mechanically fine, but seconds of model latency
after every utterance is not a conversation. Use `batch_mode=True` or transcribe
a finished recording.

Speaker labels are the model's judgement, not an acoustic decision. On a live
run against a four-turn recording with two voices, the model labelled the fourth
turn `Speaker 3` although it was the second voice again (2026-08-07). Where the
speakers are already separated — a conference records **one track per
participant** — transcribe each track with `diarize=False` and merge on the
timestamps instead. The labels earn their keep on a single mixed file.

Timestamps are the model's reading, not a forced alignment: good for navigating
a recording, not for syncing against anything.

See `examples/meeting_transcription.py` — a recording becomes speaker turns,
enters a room, and an AI channel writes the minutes.

---

## TTS Provider ABC

```python
from __future__ import annotations

from roomkit.voice.tts import AudioChunk, AudioContent, TTSProvider


class TTSProvider(ABC):
    @property
    def name(self) -> str: ...

    @property
    def default_voice(self) -> str | None: ...

    @property
    def supports_streaming_input(self) -> bool: ...

    async def synthesize(self, text, *, voice=None) -> AudioContent:
        """Batch synthesis — full text in, complete audio out."""

    async def synthesize_stream(self, text, *, voice=None) -> AsyncIterator[AudioChunk]:
        """Streaming output — yields audio chunks as they're generated."""

    async def synthesize_stream_input(self, text_stream, *, voice=None) -> AsyncIterator[AudioChunk]:
        """Streaming input — accepts async text stream, yields audio."""

    async def warmup(self) -> None:
        """Pre-load models (optional)."""

    async def close(self) -> None:
        """Release resources."""
```

### AudioChunk

```python
@dataclass
class AudioChunk:
    data: bytes
    sample_rate: int = 16000
    channels: int = 1
    format: str = "pcm_s16le"
    timestamp_ms: int | None = None
    is_final: bool = False
```

---

## ElevenLabs (Cloud API)

High-quality cloud TTS with streaming input support — starts speaking while the AI is still generating text.

```python
from __future__ import annotations

from roomkit.voice.tts.elevenlabs import ElevenLabsConfig, ElevenLabsTTSProvider

tts = ElevenLabsTTSProvider(
    config=ElevenLabsConfig(
        api_key="your-api-key",
        voice_id="21m00Tcm4TlvDq8ikWAM",    # Rachel
        model_id="eleven_multilingual_v2",
        stability=0.5,
        similarity_boost=0.75,
        style=0.0,
        use_speaker_boost=True,
        output_format="mp3_44100_128",
        optimize_streaming_latency=3,         # 0-4, higher = faster
    )
)

# List available voices
voices = await tts.list_voices()
for v in voices:
    print(f"{v['voice_id']}: {v['name']} ({v['category']})")
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `voice_id` | `"21m00Tcm4TlvDq8ikWAM"` | Voice ID (Rachel) |
| `model_id` | `"eleven_multilingual_v2"` | TTS model |
| `stability` | `0.5` | Voice stability (0–1) |
| `similarity_boost` | `0.75` | Voice similarity (0–1) |
| `style` | `0.0` | Style exaggeration (0–1) |
| `output_format` | `"mp3_44100_128"` | Output format |
| `optimize_streaming_latency` | `3` | Latency optimization level (0–4) |

**Three synthesis modes**:

- `synthesize()` — Batch: returns complete audio as base64 data URL
- `synthesize_stream()` — Streaming output: yields audio chunks via HTTP
- `synthesize_stream_input()` — Streaming input: WebSocket accepts async text, yields audio in real time

---

## SherpaOnnx TTS (Local/Offline)

Local TTS using VITS/Piper ONNX models. No API calls required.

```python
from __future__ import annotations

from roomkit.voice.tts.sherpa_onnx import SherpaOnnxTTSConfig, SherpaOnnxTTSProvider

tts = SherpaOnnxTTSProvider(
    config=SherpaOnnxTTSConfig(
        model="path/to/model.onnx",
        tokens="path/to/tokens.txt",
        data_dir="path/to/espeak-ng-data",     # For Piper models
        speaker_id=0,                           # Multi-speaker models
        speed=1.0,                              # < 1.0 = faster, > 1.0 = slower
        sample_rate=22050,
        num_threads=2,
        provider="cpu",                         # or "cuda"
    )
)
```

**Text splitting**: Automatic sentence-based chunking (max 300 chars per chunk) with short-fragment merging.

---

## Qwen3 TTS (Local/GPU, Voice Cloning)

LLM-based TTS with zero-shot voice cloning from reference audio.

```python
from __future__ import annotations

from roomkit.voice.tts.qwen3 import Qwen3TTSConfig, Qwen3TTSProvider, VoiceCloneConfig

tts = Qwen3TTSProvider(
    config=Qwen3TTSConfig(
        model_id="Qwen/Qwen3-TTS-12Hz-1.7B-Base",
        device_map="auto",
        dtype="bfloat16",
        language="English",
        voices={
            "default": VoiceCloneConfig(
                ref_audio="reference.wav",       # 3s+ clean speech
                ref_text="Transcript of the reference audio.",
            ),
            "french": VoiceCloneConfig(
                ref_audio="french_ref.wav",
                ref_text="Transcription de l'audio de reference.",
            ),
        },
        temperature=0.6,
        top_p=0.8,
        repetition_penalty=1.05,
        max_new_tokens=4096,
    )
)

# Pre-load model and encode reference audio
await tts.warmup()
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `model_id` | `"Qwen/Qwen3-TTS-12Hz-1.7B-Base"` | HuggingFace model |
| `voices` | `{}` | Voice name → `VoiceCloneConfig` mapping |
| `language` | `"English"` | Default synthesis language |
| `temperature` | `0.6` | Sampling temperature |
| `top_p` | `0.8` | Nucleus sampling probability |
| `max_new_tokens` | `4096` | Max output tokens |

**Voice cloning**: Provide a 3+ second reference WAV and its transcript. The model learns the voice characteristics at warmup time.

**Output**: Fixed 24kHz PCM.

---

## Grok TTS (Cloud API)

xAI Grok TTS with REST and bidirectional WebSocket streaming. 5 voices, 20 languages, expressive speech tags.

```python
from __future__ import annotations

from roomkit.voice.tts.grok import GrokTTSConfig, GrokTTSProvider

tts = GrokTTSProvider(
    config=GrokTTSConfig(
        api_key="your-xai-api-key",
        voice_id="eve",             # eve, ara, rex, sal, leo
        language="en",              # BCP-47 code or "auto"
        codec="pcm",               # pcm, wav, mp3, mulaw, alaw
        sample_rate=24000,          # 8000–48000
    )
)
```

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `api_key` | *(required)* | xAI API key |
| `voice_id` | `"eve"` | Voice — `eve`, `ara`, `rex`, `sal`, `leo` |
| `language` | `"en"` | BCP-47 language code or `"auto"` |
| `codec` | `"pcm"` | Output codec: `pcm`, `wav`, `mp3`, `mulaw`, `alaw` |
| `sample_rate` | `24000` | Sample rate in Hz |
| `bit_rate` | `128000` | MP3 bit rate (only used with `codec="mp3"`) |
| `base_url` | `"https://api.x.ai/v1"` | REST API base URL |
| `ws_url` | `"wss://api.x.ai/v1/tts"` | WebSocket streaming URL |
| `timeout` | `60.0` | HTTP request timeout in seconds |

### Streaming modes

- **`synthesize_stream()`** — HTTP chunked streaming from the REST endpoint
- **`synthesize_stream_input()`** — Bidirectional WebSocket: send `text.delta`/`text.done`, receive `audio.delta`/`audio.done`. Starts speaking while the AI is still generating text.

### Expressive speech tags

Grok TTS supports inline tags like `[pause]`, `[laugh]`, `[sigh]` and wrapping tags like `<whisper>`, `<soft>`, `<loud>`, `<slow>`, `<fast>`, `<high-pitch>`, `<low-pitch>`.

---

## Gradium TTS (Cloud API)

Cloud TTS with streaming input support and fine-grained voice control.

```python
from __future__ import annotations

from roomkit.voice.tts.gradium import GradiumTTSConfig, GradiumTTSProvider

tts = GradiumTTSProvider(
    config=GradiumTTSConfig(
        api_key="your-api-key",
        voice_id="default",
        region="us",
        model_name="default",
        output_format="pcm_16000",
        temperature=0.7,
        cfg_coef=2.0,               # Voice similarity (1.0–4.0)
        padding_bonus=0.0,          # Speed: negative = faster, positive = slower
        rewrite_rules="en",         # Language-specific text rewriting
    )
)
```

---

## Gemini TTS (Cloud API)

Google's generative speech models. The prompt is an *instruction*, so a
natural-language direction steers delivery — that is what `style_prompt`
exploits. 30 prebuilt voices and more than 70 documented languages.

```python
from __future__ import annotations

from roomkit.voice.tts.gemini import GeminiTTSConfig, GeminiTTSProvider

tts = GeminiTTSProvider(
    config=GeminiTTSConfig(
        api_key="your-gemini-api-key",
        model="gemini-3.1-flash-tts-preview",
        voice="Kore",                                    # 30 prebuilt voices
        language="fr-CA",                                # optional BCP-47 hint
        style_prompt="Read this in a calm voice",         # optional direction
    )
)
```

Install with `pip install roomkit[gemini]` — the same extra the Gemini AI
provider and Gemini Live use.

### Latency: not a conversational TTS

Gemini TTS trades latency for expressiveness. Measured against the live API on
2026-08-06 for a one-sentence French prompt, three runs per model:

| Model | Time to first audio | Streams incrementally |
|-------|---------------------|-----------------------|
| `gemini-3.1-flash-tts-preview` | ~5.1 s median (1.2–8.3 s) | Yes — 40 ms frames |
| `gemini-2.5-flash-preview-tts` | ~3.4 s median | No — one clip |
| `gemini-2.5-pro-preview-tts` | ~5.3 s median | No — one clip |

Seconds of dead air do not work for live turn-taking. Use Gemini TTS for
prompts, announcements, voicemail and generated audio messages; for
conversation reach for a low-latency engine (ElevenLabs, Gradium) or skip the
text round trip entirely with
[Gemini Live speech-to-speech](realtime-voice-providers.md).

The default model is the only one that streams as it generates, which is why it
is the default despite a higher median: playback can start on the first frame
instead of waiting for the whole clip.

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `api_key` | *(required)* | Gemini API key (`GEMINI_API_KEY`) |
| `model` | `"gemini-3.1-flash-tts-preview"` | One of the three models above |
| `voice` | `"Kore"` | Prebuilt voice name |
| `language` | `None` | BCP-47 hint; unset, the model infers it from the text |
| `style_prompt` | `None` | Delivery direction. Not an API field — written as a labelled `Delivery direction:` line above the transcript in the same prompt |
| `timeout` | `120.0` | Per-request timeout in seconds |

### Output format

Always 24 kHz, 16-bit, mono PCM — fixed by the service. The request accepts a
`sample_rate` field but the service ignores it, so the provider does not expose
the knob; attach a [resampler stage](resampler.md) when the transport needs
another rate.

### Voices

`GeminiTTSProvider.available_voices()` returns the 30 prebuilt voices as
`VoiceInfo` records — the same catalog Gemini Live native audio draws from, so a
voice chosen for one works in the other.

```python
for v in GeminiTTSProvider.available_voices():
    print(v.id, "—", v.description)   # Kore — Firm
```

### Expressive audio tags

The delivery is steerable from the text itself. Bracketed cues placed inline in
the transcript are *performed*, not read out — this is what a conventional
concatenative engine cannot do, and what the latency buys:

```python
await tts.synthesize("[laughs] Okay, that one was actually funny.")
await tts.synthesize("[whispers] Can you keep a secret? [excitedly] We shipped!")
```

Two kinds, both inline: non-verbal sounds (`[laughs]`, `[sighs]`, `[gasp]`,
`[cough]`) and delivery modifiers (`[whispers]`, `[shouting]`, `[excitedly]`,
`[bored]`, `[very slowly]`, `[singing]`, `[asmr]`). Google documents no closed
list — any descriptive cue is interpreted — so listen to an unusual one before
relying on it: an unrecognised cue can be spoken aloud instead of performed.
With a non-English transcript, keep the tags in English.

The API has no style field: `model`, `input`, `stream`, `response_format` and
`speech_config` (voice, language) are all it takes, and style is only
expressible inside `input`. `style_prompt` is RoomKit's sugar over exactly
that — it writes a labelled `Delivery direction:` line above the `Transcript:`
label in the same string, which is what stops the model from reciting the
direction along with the words.

Tags and `style_prompt` are therefore different tools: a tag steers a word or a
phrase from inside the transcript, `style_prompt` steers the whole utterance
from the line above it.
Google's own guidance also frames a full direction in three parts — audio
profile (who is speaking), scene (where, what mood), and director's notes
(style, accent, pacing) — which is the shape `style_prompt` is for. See
Google's [prompt guide](https://aistudio.google.com/learn/gemini-tts-prompt-guide-with-tags).

Tags travel through a room like any other text, so anything written into a room
attached to a voice channel is performed: `examples/gemini_tts_room.py` is a
room that speaks what you type at the CLI.

### SSML is not an input mode here

SSML belongs to Cloud Text-to-Speech, which accepts it in a dedicated field
(`SynthesisInput(ssml=...)`). The Gemini API endpoint this provider calls has a
single free-text `input` field and no `ssml` field, and Google's Gemini-TTS page
documents prompting rather than SSML.

Sent anyway, the markup is not ignored: the model reads it as an instruction and
follows the *intent*, not the timing. Longest silence produced, measured against
the live API on 2026-08-07, three runs each:

| Asked for | Runs | Median |
|-----------|------|--------|
| `<break time="1500ms"/>` | 1.60 s, 2.46 s, 2.08 s | 2.08 s |
| `<break time="5000ms"/>` | 3.36 s, 5.70 s, 4.72 s | 4.72 s |
| `[long pause]`, no duration given | 3.48 s, 4.02 s, 4.46 s | 4.02 s |

Directionally right, never exact — the same request drifts by seconds between
identical runs, and a bracketed cue gets you the same effect without pretending
to a contract. Anything that has to line up with something else (a beep, a
prompt, a recording) needs the silence assembled in the outbound audio instead,
not requested from the model.

### Streaming

`synthesize_stream()` forwards audio deltas as they arrive.
`synthesize_stream_input()` is **not** supported: the API takes a complete
prompt, so there is no seam for token deltas. A `VoiceChannel` detects this and
delivers the finished reply through `synthesize_stream()` instead.

Gemini 3.1 TTS remains a preview model. Google documents rare cases where it
reads prompt directions aloud or returns a transient HTTP 500 instead of audio;
keep prompts explicit, split outputs longer than a few minutes, and apply retry
at the calling workflow boundary when a failed generation is safe to repeat.

See `examples/gemini_tts.py`.

---

## NeuTTS (Local/GPU, Voice Cloning)

GGUF-quantized LLM-based TTS with native streaming and voice cloning.

```python
from __future__ import annotations

from roomkit.voice.tts.neutts import NeuTTSConfig, NeuTTSProvider, NeuTTSVoiceConfig

tts = NeuTTSProvider(
    config=NeuTTSConfig(
        backbone_repo="neuphonic/neutts-nano-french-q8-gguf",
        codec_repo="neuphonic/neucodec",
        device="cpu",                           # or "cuda"
        voices={
            "default": NeuTTSVoiceConfig(
                ref_audio="reference.wav",       # 3-15s, 16kHz mono
                ref_text="Transcript of reference audio.",
            ),
        },
        streaming_pre_buffer=2,                  # Chunks to buffer before yielding
    )
)
```

**Pre-buffering**: On CPU, accumulates 2 chunks (~1 second) before yielding to prevent playback underruns when inference is slower than real-time.

**Output**: Fixed 24kHz PCM.

---

## TTS Filters

Filters clean AI-generated text before it reaches the TTS provider. Essential for removing reasoning markers, annotations, or bracketed instructions.

### StripInternalTags

Removes `[internal]...[/internal]` and `[internal: ...]` blocks — useful when the AI includes reasoning that shouldn't be spoken.

```python
from __future__ import annotations

from roomkit.voice.tts.filters import StripInternalTags

f = StripInternalTags()

# Non-streaming (full text)
clean = f("[internal]Let me think about this...[/internal] Here's what I found.")
# → "Here's what I found."

# Streaming (token by token)
for token in ["[internal", "]thinking[/", "internal] The answer", " is 42."]:
    result = f.feed(token)
    if result:
        print(result, end="")
print(f.flush())
# → "The answer is 42."
```

### StripBrackets

Removes all `[...]` bracketed content — catches `[laughs]`, `[pause]`, `[Respond in French]`, etc.

```python
from __future__ import annotations

from roomkit.voice.tts.filters import StripBrackets

f = StripBrackets()
clean = f("Sure [laughs] I can help [pause] with that.")
# → "Sure  I can help  with that."
```

### Using Filters with Streaming TTS

```python
from __future__ import annotations

from roomkit.voice.tts.filters import StripInternalTags, filtered_stream


async def ai_token_stream():
    """Simulated AI output with internal reasoning."""
    for token in ["[internal]", "reasoning", "[/internal]", " Hello", " there!"]:
        yield token


# Wrap the token stream through a filter before TTS
clean_stream = filtered_stream(ai_token_stream(), StripInternalTags())

async for chunk in tts.synthesize_stream_input(clean_stream, voice="default"):
    # Audio chunks without the internal reasoning
    transport.send_audio(session, chunk)
```

### Sentence Splitter

Buffers streaming tokens and yields complete sentences — prevents unnatural pauses from very short fragments.

```python
from __future__ import annotations

from roomkit.voice.tts.sentence_splitter import split_sentences

# Buffer tokens until sentence boundaries
async for sentence in split_sentences(ai_token_stream(), min_chunk_chars=20):
    async for chunk in tts.synthesize_stream(sentence, voice="default"):
        transport.send_audio(session, chunk)
```

---

## Choosing a Provider

| Provider | Type | Streaming | Latency | Cost | Best For |
|----------|------|-----------|---------|------|----------|
| **Deepgram** | Cloud STT | Yes | Low | Per-minute | Production real-time transcription |
| **Gradium** | Cloud STT | Yes | Low | Per-minute | Real-time with server-side VAD |
| **SherpaOnnx** | Local STT | Transducer only | Medium | Free | Privacy, offline, edge |
| **Qwen3 ASR** | Local STT | vLLM only | Medium | Free | GPU-accelerated, multilingual |
| **ElevenLabs** | Cloud TTS | Yes + input | Low | Per-character | Highest voice quality |
| **Grok** | Cloud TTS | Yes + input | Low | Per-character | Expressive tags, 20 languages |
| **Gradium** | Cloud TTS | Yes + input | Low | Per-character | Real-time with voice control |
| **SherpaOnnx** | Local TTS | Yes | Medium | Free | Privacy, offline, VITS/Piper |
| **Qwen3 TTS** | Local TTS | Post-gen | Medium | Free | Voice cloning, GPU |
| **NeuTTS** | Local TTS | GGUF only | Medium | Free | Voice cloning, GGUF quantized |

## Using with VoiceChannel

```python
from __future__ import annotations

from roomkit.channels import VoiceChannel
from roomkit.voice.backends.mock import MockVoiceBackend
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.stt.deepgram import DeepgramConfig, DeepgramSTTProvider
from roomkit.voice.tts.elevenlabs import ElevenLabsConfig, ElevenLabsTTSProvider

stt = DeepgramSTTProvider(config=DeepgramConfig(api_key="..."))
tts = ElevenLabsTTSProvider(config=ElevenLabsConfig(api_key="..."))

voice = VoiceChannel(
    "voice-main",
    stt=stt,
    tts=tts,
    backend=MockVoiceBackend(),
    pipeline=AudioPipelineConfig(),
)

kit.register_channel(voice)
```

## Testing with Mocks

```python
from __future__ import annotations

from roomkit.voice.stt.mock import MockSTTProvider
from roomkit.voice.tts.mock import MockTTSProvider

stt = MockSTTProvider(transcripts=["Hello", "How are you?"], streaming=False)
tts = MockTTSProvider(voice="mock-voice")

# After usage:
assert len(stt.calls) == 1                # Audio inputs received
assert len(tts.calls) == 1                # Synthesis requests made
assert tts.calls[0]["text"] == "Hello!"   # Text synthesized
```
