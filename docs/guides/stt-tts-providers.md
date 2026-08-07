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

    async def transcribe(self, audio) -> TranscriptionResult:
        """Batch transcription — send all audio, get full text."""

    async def transcribe_stream(self, audio_stream) -> AsyncIterator[TranscriptionResult]:
        """Streaming transcription — get partial results in real time."""

    async def warmup(self) -> None:
        """Pre-load models (optional)."""

    async def close(self) -> None:
        """Release resources."""
```

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
| `model` | `"nova-3"` | Model name |
| `language` | `"en"` | Language code |
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
        style_prompt="Lis ce texte d'une voix calme",     # optional direction
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
| `style_prompt` | `None` | Delivery direction, separated from a labelled transcript |
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
