# Audio Pipeline Stages

RoomKit's `AudioPipeline` processes audio between the voice backend and STT/TTS through 11 pluggable stages. Each stage is optional — configure only what you need.

## Pipeline Flow

```
Inbound:  Backend → [Resampler] → [Recorder] → [DTMF] → [AEC] → [AGC] → [Denoiser] → [VAD] → [Diarization]
Outbound: [PostProcessors] → [Recorder] → AEC.feed_reference → [Resampler]
```

## Configuration

All stages are configured via `AudioPipelineConfig`:

```python
from __future__ import annotations

from roomkit.channels import VoiceChannel
from roomkit.voice.pipeline import AudioPipelineConfig

pipeline = AudioPipelineConfig(
    vad=vad_provider,
    aec=aec_provider,
    agc=agc_provider,
    denoiser=denoiser_provider,
    diarization=diarization_provider,
    dtmf=dtmf_detector,
    recorder=recorder,
    resampler=resampler,
    postprocessors=[normalizer],
    turn_detector=turn_detector,
    backchannel_detector=backchannel_detector,
    debug_taps=debug_taps,
    telemetry=telemetry_provider,
    interruption=interruption_config,
)

voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend, pipeline=pipeline)
```

### Running the inbound chain off the event loop

By default every inbound frame runs the full stage chain on the thread
that delivered it — usually the event loop. That caps concurrent sessions
at one core, and one slow stage delays every other session's audio and
all message traffic.

`inbound_dsp_threads` moves the chain onto a small thread pool:

```python
pipeline = AudioPipelineConfig(
    vad=vad_provider,
    denoiser=denoiser_provider,
    inbound_dsp_threads=4,   # size to your core count
)
```

Guarantees and behaviour:

- **Frames of one session stay strictly FIFO** — a per-stream queue is
  drained by at most one worker at a time. Only *which sessions* run in
  parallel is decided by the pool, so the RFC §12 stage order is
  untouched.
- **The native stages release the GIL** (SpeexDSP, WebRTC AEC, RNNoise,
  sherpa-onnx, NumPy), so pool threads deliver real multi-core
  parallelism — measured 3.9× on 4 workers in the stress bench.
- **Backpressure is per stream and bounded**: when a stream's queue is
  full the oldest frame is dropped and counted (late audio is worthless
  audio; a stalled consumer never grows memory).
- Applies to both `VoiceChannel` and `RealtimeVoiceChannel`. Unset keeps
  today's inline processing.

---

## Stage: Resampler

Converts audio between transport and internal formats. Auto-created when `AudioPipelineContract` specifies different rates.

```python
from roomkit.voice.pipeline.resampler import LinearResamplerProvider, SincResamplerProvider

# Fast, low-quality (good for voice)
resampler = LinearResamplerProvider()

# High-quality (better for music/analysis)
resampler = SincResamplerProvider()
```

| Provider | Quality | Latency | Use Case |
|----------|---------|---------|----------|
| `LinearResamplerProvider` | Good | Lowest | Voice conversations |
| `SincResamplerProvider` | Best | Higher | Audio analysis, music |

The pipeline uses separate resampler instances for inbound, outbound, and AEC reference paths to avoid state conflicts.

---

## Stage: Audio Recorder

Records inbound and/or outbound audio to files. See the [WAV File Recorder guide](wav-file-recorder.md) for the `WavFileRecorder` implementation.

```python
from roomkit.voice.pipeline.recorder import AudioRecorder, RecordingConfig, RecordingMode, RecordingChannelMode

config = RecordingConfig(
    mode=RecordingMode.BOTH,                 # INBOUND_ONLY, OUTBOUND_ONLY, BOTH
    channels=RecordingChannelMode.STEREO,    # MIXED, SEPARATE, STEREO
    trigger=RecordingTrigger.ALWAYS,         # ALWAYS, SPEECH_ONLY
    format="wav",
    storage="/recordings",
    storage_encrypted_at_rest=True,          # storage encryption is a deployment guarantee
)
```

| Mode | Description |
|------|-------------|
| `INBOUND_ONLY` | Record mic audio only |
| `OUTBOUND_ONLY` | Record TTS audio only |
| `BOTH` | Record both directions |

| Channel Mode | Description |
|-------------|-------------|
| `MIXED` | Blend inbound/outbound into single stream |
| `SEPARATE` | Write separate files per direction |
| `STEREO` | L/R channels (inbound=left, outbound=right) |

Hooks: `ON_RECORDING_STARTED`, `ON_RECORDING_STOPPED`.

---

## Stage: DTMF Detection

Detects touch-tone digits (0-9, \*, #, A-D) from audio. Runs **before** AEC and denoiser to preserve tone frequencies.

```python
from roomkit.voice.pipeline.dtmf import DTMFDetector, DTMFEvent
```

The detector emits `DTMFEvent(digit, duration_ms, confidence)` and fires the `ON_DTMF` hook.

!!! tip "IVR Menus"
    DTMF detection enables IVR (Interactive Voice Response) menus. Combine with a `BEFORE_BROADCAST` hook to route calls based on digit presses.

**Inband vs signaling**: Inband DTMF reads tones from the audio stream. SIP backends can also receive DTMF via out-of-band signaling (SIP INFO / RFC 4733). Use `VoiceCapability.DTMF_INBAND` and `DTMF_SIGNALING` to indicate which your backend supports.

**Outbound DTMF**: To *send* DTMF digits to the remote party (e.g., an AI agent navigating an IVR menu), use `VoiceChannel.send_dtmf(session, digit, duration_ms=160)`. This sends RFC 4733 telephone-events via the SIP or RTP backend. See the [SIP backend guide](sip-backend.md#outbound-sending) for details.

### Redaction

DTMF digits carry card numbers, PINs and account numbers. `DTMFRedaction` masks
them wherever the framework itself exposes a digit:

```python
from roomkit.voice.pipeline import AudioPipelineConfig, DTMFRedaction

pipeline = AudioPipelineConfig(
    vad=vad,
    dtmf=detector,
    dtmf_redaction=DTMFRedaction(keep_first=4, keep_last=4),
)
```

`mask()` is also usable directly on a sequence your own code accumulates:

```python
DTMFRedaction(keep_first=4, keep_last=4).mask("4111111111111111")
# '4111********1111'
```

| Field | Default | Description |
|-------|---------|-------------|
| `enabled` | `True` | The object exists to turn masking on |
| `keep_first` | `0` | Leading digits left in the clear |
| `keep_last` | `0` | Trailing digits left in the clear |
| `mask_char` | `"*"` | Character substituted for each masked digit |

The defaults mask everything: a redaction that leaks the head of a PIN by
default would be a worse trap than none at all. A sequence shorter than the
digits it would keep in the clear is masked entirely — keeping edges gives
context on a long number, not a peephole onto a short secret.

What redaction covers, and what it deliberately does not:

- **Masked** — `frame.metadata["dtmf"]["digit"]`, which travels to recorders,
  debug taps and logs, and gains `"redacted": True`.
- **Masked** — `DTMFDetectedEvent.redacted_digit`, for anything that logs,
  stores or transcribes from a hook. It equals `digit` when no redaction is
  configured, so a handler can use it unconditionally.
- **Not masked** — `DTMFDetectedEvent.digit`. The `ON_DTMF` hook is how an IVR
  reads the digits it exists to collect; masking it would remove the feature
  rather than protect it.

---

## Stage: AEC (Acoustic Echo Cancellation)

Removes speaker audio echoing back through the microphone. Essential for speakerphone and speaker+mic setups.

```python
from roomkit.voice.pipeline.aec import SpeexAECProvider, WebRTCAECProvider

speex_aec = SpeexAECProvider(
    sample_rate=16000,
    frame_size=320,       # 20ms at 16kHz
    filter_length=1024,   # Echo tail length in samples
)

webrtc_aec = WebRTCAECProvider(
    sample_rate=24000,
    channels=1,
    stream_delay_ms=0,    # 0 lets WebRTC estimate the delay
)
```

| Provider | Library | Notes |
|----------|---------|-------|
| `SpeexAECProvider` | libspeexdsp (ctypes) | Split API with internal ring buffer |
| `WebRTCAECProvider` | aec-audio-processing (pip) | WebRTC-based AEC |

**How it works**: The pipeline automatically feeds TTS playback audio as the reference signal via `feed_reference(frame, stream)` on the outbound path. The `process(frame, stream)` method on the inbound path uses that stream's reference to subtract echo. Both carry the same key, because each stream owns its echo canceller — in a conference every lane hears a different mix.

The capture and reference signals must have the same sample rate, channel count,
sample width, and timeline. RoomKit records the post-resampler capture format per
stream and converts the reference to that exact format. WebRTC and Speex accept
16-bit PCM; an unexpected format is bypassed with a warning instead of being
silently misinterpreted.

AEC activation is also isolated per stream and playback source. When playback
ends, RoomKit bypasses the stream but preserves the converged adaptive filter;
the filter is reset only when the voice session ends. For a transport-owned AEC
path such as `LocalAudioBackend(aec=aec)`, the reference follows audio actually
rendered by the device. Once playback starts, hardware silence inserted during
an underrun remains part of the reference timeline so capture and render do not
drift apart. RoomKit prevents the pipeline from feeding that same reference a
second time.

`WebRTCAECProvider` processes 10 ms blocks internally and accepts arbitrary
input chunk sizes. Leave `stream_delay_ms=0` initially so WebRTC can estimate
alignment. Set a non-zero value only when you have measured a stable
speaker-to-microphone delay; an incorrect fixed delay can reduce cancellation.

`enable_ns` and `enable_agc` remain available on `WebRTCAECProvider` for compact
integrations, and their capture effects now remain active while AEC is bypassed.
For explicit pipeline ordering and independent tuning, prefer
`WebRTCNoiseSuppressorProvider` plus `SimpleAGCProvider` as shown below.

!!! note "Capability-Aware Skipping"
    When the backend declares `VoiceCapability.NATIVE_AEC`, the pipeline skips the AEC stage — the backend handles echo cancellation natively.

---

## Stage: AGC (Automatic Gain Control)

Normalizes audio volume levels. Useful when users have different microphone
volumes. AGC is not noise cancellation: it adjusts amplitude, while the next
denoiser stage removes fans, traffic, and other background sound.

```python
from roomkit.voice.pipeline import AGCConfig, AudioPipelineConfig, SimpleAGCProvider

# Supplying only agc_config selects the built-in SimpleAGCProvider.
pipeline = AudioPipelineConfig(
    agc_config=AGCConfig(
        target_level_dbfs=-3.0,     # Target output level
        max_gain_db=30.0,           # Maximum gain applied
        attack_ms=10.0,             # Fast attack for sudden volume changes
        release_ms=100.0,           # Slower release for natural decay
    ),
)

# Equivalent explicit construction, useful when sharing the provider instance:
agc = SimpleAGCProvider(AGCConfig(target_level_dbfs=-12.0))
pipeline = AudioPipelineConfig(agc=agc)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `target_level_dbfs` | `-3.0` | Target output level in dBFS |
| `max_gain_db` | `30.0` | Maximum gain to apply |
| `attack_ms` | `10.0` | Attack time (how fast gain increases) |
| `release_ms` | `100.0` | Release time (how fast gain decreases) |

`SimpleAGCProvider` keeps gain history per stream, does not amplify frames below
its `-60 dBFS` silence floor, and applies a peak limiter to prevent clipping.
Each processed frame includes `metadata.gain_applied_db`. Provider-specific
`silence_threshold_dbfs` and `min_gain_db` overrides can be placed in
`AGCConfig.metadata`.

!!! note
    Auto-skipped when backend has `VoiceCapability.NATIVE_AGC`.

---

## Stage: Denoiser

Reduces background noise (fans, traffic, keyboard clicks).

```python
from roomkit.voice.pipeline.denoiser import (
    RNNoiseDenoiserProvider,
    WebRTCNoiseSuppressorProvider,
)

webrtc_ns = WebRTCNoiseSuppressorProvider(sample_rate=24000)
rnnoise = RNNoiseDenoiserProvider(sample_rate=24000)
```

| Provider | Library | Notes |
|----------|---------|-------|
| `WebRTCNoiseSuppressorProvider` | aec-audio-processing (pip) | Continuous WebRTC NS; 10 ms internal frames |
| `RNNoiseDenoiserProvider` | librnnoise (system) | Mono PCM16 at 16, 24, or 48 kHz |
| `SherpaOnnxDenoiserProvider` | sherpa-onnx (pip) | ONNX models, configurable context and silence threshold |
| `AICousticsDenoiserProvider` | aic-sdk (pip) | Quail neural enhancement, Voice Focus, ~2 ms/frame |

For SherpaOnnx denoiser tuning, see the [sherpa-onnx guide](sherpa-onnx.md). For ai|coustics Quail setup, see the [ai|coustics denoiser guide](aicoustics-denoiser.md).

All built-in denoisers validate their PCM format before native processing and
keep recurrent state per stream. WebRTC NS, RNNoise, and ai|coustics accept
arbitrary caller chunk sizes; after the first irregular fragment they use a
fixed one-native-block delay (10 ms for WebRTC NS and RNNoise) so every input
byte has exactly one output byte. This avoids the duplicate/raw fragment pattern
that can otherwise leak noise into VAD and STT.

---

## Stage: VAD (Voice Activity Detection)

Detects speech start and end. This is the most critical pipeline stage — it drives STT segmentation and interruption handling.

```python
from roomkit.voice.pipeline.vad import SherpaOnnxVADProvider, EnergyVADProvider, VADConfig
from roomkit.voice.pipeline import AudioPipelineConfig

vad = SherpaOnnxVADProvider(model_path="silero_vad.onnx")

pipeline = AudioPipelineConfig(
    vad=vad,
    vad_config=VADConfig(
        silence_threshold_ms=500,      # Silence duration to end speech
        speech_pad_ms=300,             # Padding around speech segments
        min_speech_duration_ms=250,    # Minimum utterance length
    ),
)
```

| Provider | Method | Notes |
|----------|--------|-------|
| `SherpaOnnxVADProvider` | Neural network (Silero) | Accurate, recommended |
| `EnergyVADProvider` | RMS energy threshold | Simple, fast, less accurate |

### VAD Events

| Event | Meaning |
|-------|---------|
| `SPEECH_START` | User started speaking |
| `SPEECH_END` | User stopped speaking (includes accumulated audio) |
| `SILENCE` | Silence detected |
| `AUDIO_LEVEL` | Periodic audio level update |

Hooks: `ON_SPEECH_START`, `ON_SPEECH_END`, `ON_VAD_SILENCE`, `ON_VAD_AUDIO_LEVEL`.

---

## Stage: Speaker Diarization

Identifies **who** is speaking. Only processes frames during active speech (requires VAD).

```python
from roomkit.voice.pipeline.diarization import SherpaOnnxDiarizationProvider

diarization = SherpaOnnxDiarizationProvider(model_path="speaker_model.onnx")
```

Returns `DiarizationResult(speaker_id, confidence, is_new_speaker)` and fires `ON_SPEAKER_CHANGE` when the speaker changes.

!!! tip
    Diarization is useful for multi-party calls where you need to attribute transcriptions to specific speakers.

---

## Stage: PostProcessor

Transforms outbound TTS audio before playback. Runs after TTS, before recorder and AEC reference feeding.

```python
from __future__ import annotations

from roomkit.voice.pipeline.postprocessor import AudioPostProcessor
from roomkit.voice.base import AudioFrame


class VolumeNormalizer(AudioPostProcessor):
    @property
    def name(self) -> str:
        return "volume_normalizer"

    def process(self, frame: AudioFrame, stream: str) -> AudioFrame:
        # Apply volume normalization
        return frame

    def reset(self, stream: str) -> None:
        # Drop this stream's state — the stream has ended
        ...


pipeline = AudioPipelineConfig(postprocessors=[VolumeNormalizer()])
```

Multiple postprocessors are applied in order. Use cases: volume normalization, audio watermarking, effects.

### Writing a stage: keep your state per stream

Every stage method takes a `stream` key — an opaque identifier for the audio
stream the frame belongs to. One pipeline serves many streams: a bridged voice
channel has one per session, a conference one per lane. **Your stage must key
its state on it.** Sharing state between streams is what makes one speaker's
silence close another speaker's utterance.

```python
from dataclasses import dataclass, field


@dataclass
class _StreamState:
    # Only what process() mutates. Immutable config stays on the instance.
    buffer: bytearray = field(default_factory=bytearray)


class MyStage(AudioPostProcessor):
    def __init__(self) -> None:
        self._streams: dict[str, _StreamState] = {}

    def process(self, frame: AudioFrame, stream: str) -> AudioFrame:
        st = self._streams.setdefault(stream, _StreamState())
        st.buffer.extend(frame.data)
        return frame

    def reset(self, stream: str) -> None:
        self._streams.pop(stream, None)

    def close(self) -> None:
        self._streams.clear()
```

`stream` has no default on purpose: a stage that quietly ignored it would still
satisfy the interface while mixing speakers together. If your stage wraps a
native SDK, note that the model and the adaptive state usually live in the same
object — one stream then means one native instance, so release it in **both**
`reset(stream)` and `close()`. A leak there is C memory, not a red test.

RoomKit ships a conformance check you can run against your own stage:

```python
from tests.voice.pipeline.stream_conformance import (
    assert_stage_keeps_state_per_stream,
)


def test_my_stage_is_per_stream():
    assert_stage_keeps_state_per_stream(MyStage, make_frame)
```

---

## Stage: Turn Detection

Determines if the user has finished their turn. Integrated post-STT in VoiceChannel (not in the core pipeline engine).

```python
from roomkit.voice.pipeline.turn import TurnDetector, TurnContext, TurnDecision
```

The detector receives `TurnContext` with transcript, silence duration, conversation history, and audio data. Returns `TurnDecision(is_complete, confidence, reason, suggested_wait_ms)`.

Hooks: `ON_TURN_COMPLETE`, `ON_TURN_INCOMPLETE`.

See the [Smart Turn Detection guide](smart-turn.md) for details.

---

## Stage: Backchannel Detection

Classifies short utterances as backchannels ("uh-huh", "yeah") vs real interruptions. Used by the SEMANTIC interruption strategy.

```python
from roomkit.voice.pipeline.backchannel import BackchannelDetector, BackchannelContext, BackchannelDecision
```

See the [Voice Interruption guide](voice-interruption.md) for details on SEMANTIC strategy and backchannel detection.

---

## Capability-Aware Skipping

Backends declare capabilities via `VoiceCapability` flags:

| Flag | Effect |
|------|--------|
| `NATIVE_AEC` | Pipeline skips AEC stage |
| `NATIVE_AGC` | Pipeline skips AGC stage |
| `INTERRUPTION` | Backend supports `cancel_audio()` |
| `BARGE_IN` | Backend handles barge-in detection |
| `DTMF_INBAND` | DTMF detected from audio stream |
| `DTMF_SIGNALING` | DTMF sent and received via signaling (RFC 4733) |

```python
from roomkit.voice.base import VoiceCapability

# A backend that provides native AEC — pipeline auto-skips AEC stage
backend._capabilities = VoiceCapability.INTERRUPTION | VoiceCapability.NATIVE_AEC
```

---

## Frame Metadata

Each stage annotates processed frames with metadata:

| Stage | Metadata Key | Value |
|-------|-------------|-------|
| Resampler | `original_sample_rate`, `original_channels` | Original format info |
| DTMF | `dtmf` | `{"digit": str, "duration_ms": float}` |
| AEC | `aec` | Provider name |
| AGC | `agc` | Provider name |
| Denoiser | `denoiser` | Provider name |
| VAD | `vad` | `{"type": VADEventType, "confidence": float}` |
| VAD (state) | `vad_is_speech`, `vad_speech_end` | `True` during speech / at boundary |
| Diarization | `diarization` | `{"speaker_id": str, "confidence": float}` |

Access metadata on processed frames:

```python
pipeline.on_processed_frame(lambda session, frame: print(frame.metadata))
```

---

## Pipeline Debug Taps

Insert recording/analysis taps at specific pipeline points:

```python
from roomkit.voice.pipeline import AudioPipelineConfig, PipelineDebugTaps

taps = PipelineDebugTaps(
    inbound_tap=my_recorder,    # Tap after inbound processing
    outbound_tap=my_recorder,   # Tap after outbound processing
)

pipeline = AudioPipelineConfig(debug_taps=taps)
```

---

## Pipeline Lifecycle

All stage providers follow the same lifecycle contract:

| Method | When Called | Purpose |
|--------|-----------|---------|
| `process(frame)` | Every audio frame | Core processing |
| `reset()` | Session start | Clear internal state |
| `close()` | Shutdown | Release resources |

The pipeline calls `reset()` on all stages when a voice session becomes active, and `close()` on shutdown.

---

## Error Handling

Every stage wraps processing in try/except. A failed stage does not crash the pipeline — the frame passes through unchanged (graceful degradation):

```
AEC error → frame passes through unprocessed → denoiser still runs → VAD still works
```

---

## Full Example

```python
from __future__ import annotations

from roomkit.channels import VoiceChannel
from roomkit.voice.interruption import InterruptionConfig, InterruptionStrategy
from roomkit.voice.pipeline import AudioPipelineConfig
from roomkit.voice.pipeline.aec import SpeexAECProvider
from roomkit.voice.pipeline.denoiser import RNNoiseDenoiserProvider
from roomkit.voice.pipeline.vad import SherpaOnnxVADProvider, VADConfig

# Configure stages
vad = SherpaOnnxVADProvider(model_path="silero_vad.onnx")
aec = SpeexAECProvider(sample_rate=16000, frame_size=320, filter_length=1024)
denoiser = RNNoiseDenoiserProvider()

# Build pipeline
pipeline = AudioPipelineConfig(
    vad=vad,
    vad_config=VADConfig(silence_threshold_ms=500, min_speech_duration_ms=250),
    aec=aec,
    denoiser=denoiser,
    interruption=InterruptionConfig(
        strategy=InterruptionStrategy.CONFIRMED,
        min_speech_ms=300,
    ),
)

# Create voice channel
voice = VoiceChannel(
    "voice",
    stt=stt,
    tts=tts,
    backend=backend,
    pipeline=pipeline,
)
```

Processing order for this configuration:

1. **Resampler** (auto, if format mismatch)
2. **AEC** — remove echo using TTS playback reference
3. **Denoiser** — reduce background noise
4. **VAD** — detect speech start/end
5. STT receives clean, segmented audio
