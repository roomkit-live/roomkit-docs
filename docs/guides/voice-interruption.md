# Voice Interruption Handling

RoomKit provides four interruption strategies to control what happens when a user speaks while the AI is talking. This guide covers configuration, backchannel detection, and practical patterns for each strategy.

## Quick Start

```python
from __future__ import annotations

from roomkit.channels import VoiceChannel
from roomkit.voice.interruption import InterruptionConfig, InterruptionStrategy

voice = VoiceChannel(
    "voice",
    stt=stt,
    tts=tts,
    backend=backend,
    interruption=InterruptionConfig(
        strategy=InterruptionStrategy.CONFIRMED,
        min_speech_ms=300,
    ),
)
```

## Strategy Comparison

| Strategy | Behavior | Use Case | False Positives |
|----------|----------|----------|-----------------|
| **IMMEDIATE** | Cancel TTS on any speech detection | IVR, command interfaces | Highest |
| **CONFIRMED** | Wait for `min_speech_ms` before interrupting | General conversation | Low |
| **SEMANTIC** | Use backchannel detection to distinguish "uh-huh" from real interruptions | Customer support, therapy | Lowest |
| **DISABLED** | Never interrupt — queue user speech until playback ends | Announcements, legal disclaimers | None |

## Configuration

```python
from roomkit.voice.interruption import InterruptionConfig, InterruptionStrategy

config = InterruptionConfig(
    strategy=InterruptionStrategy.CONFIRMED,
    min_speech_ms=300,
    allow_during_first_ms=0,
    flush_partial_tts=True,
    keep_partial_transcript=True,
    backchannel_detector=None,
)
```

| Field | Default | Description |
|-------|---------|-------------|
| `strategy` | `CONFIRMED` | Which interruption strategy to use |
| `min_speech_ms` | `300` | Minimum speech duration (ms) before confirming interruption (CONFIRMED strategy) |
| `allow_during_first_ms` | `0` | Grace period: don't interrupt during the first N ms of playback |
| `flush_partial_tts` | `True` | Discard audio already handed to the backend. `False` lets the current utterance finish while the user's speech is processed alongside it |
| `keep_partial_transcript` | `True` | Record what the room actually heard as an internal event when the bot is cut off |
| `backchannel_detector` | `None` | Required for SEMANTIC strategy |

### What an interruption leaves behind

The AI's response event is written when the text is *produced*, not when it is
heard, so a bot cut off after two words still has its whole line in the
timeline. With `keep_partial_transcript` on, the interruption records what the
room actually heard as an `internal`-visibility event:

```python
{
    "interrupted": True,
    "played_ms": 1840,
    "voice_session_id": "...",
    # "played_percentage": 46.0   — only when the utterance's duration is known
}
```

The full text is stored rather than a truncated guess: `played_ms` says how far
playback got, and cutting the string at the same proportion would invent a word
boundary that TTS timing does not guarantee.

## Strategy Details

### IMMEDIATE

Cancels TTS playback the instant any speech is detected. Best for systems where responsiveness matters more than naturalness.

```python
voice = VoiceChannel(
    "voice",
    stt=stt, tts=tts, backend=backend,
    interruption=InterruptionConfig(
        strategy=InterruptionStrategy.IMMEDIATE,
        allow_during_first_ms=200,  # Prevent echo-triggered interruptions
    ),
)
```

!!! tip
    Use `allow_during_first_ms` to add a grace period at the start of playback. This prevents the speaker's own audio echoing back and triggering a false interruption.

### CONFIRMED

Waits until the user has been speaking for at least `min_speech_ms` before confirming the interruption. Filters out brief noise and accidental sounds.

The evaluation happens twice. At speech onset no duration has accumulated yet,
so the first look can only say "not yet" — the segment is held as possible echo
and a second look is taken once the speech has had `min_speech_ms` to sustain.
If the speech stopped in between, it was a blip and nothing is interrupted. The
confirmation logs at INFO:

```
Barge-in confirmed after 300ms of sustained speech (session <id>)
```

That deferral is audible in one place: with a streaming STT provider, capture
starts at the confirmation, so roughly the first `min_speech_ms` of the
utterance does not reach the streaming transcript. The batch fallback receives
the whole segment, so the text is not lost — but lower `min_speech_ms` if you
need the leading words on the streaming path.

```python
voice = VoiceChannel(
    "voice",
    stt=stt, tts=tts, backend=backend,
    interruption=InterruptionConfig(
        strategy=InterruptionStrategy.CONFIRMED,
        min_speech_ms=300,  # Require 300ms of continuous speech
    ),
)
```

### SEMANTIC

Uses a `BackchannelDetector` to distinguish backchannels ("uh-huh", "yeah", "ok") from real interruptions. Falls back to CONFIRMED if no detector is configured.

```python
voice = VoiceChannel(
    "voice",
    stt=stt, tts=tts, backend=backend,
    interruption=InterruptionConfig(
        strategy=InterruptionStrategy.SEMANTIC,
        backchannel_detector=my_detector,
    ),
)
```

### DISABLED

Never interrupts. Speech that arrives while the assistant is talking is
**queued, not discarded**: the assistant finishes its turn, and the held
utterance is then processed as if it had just been spoken.

```python
voice = VoiceChannel(
    "voice",
    stt=stt, tts=tts, backend=backend,
    interruption=InterruptionConfig(
        strategy=InterruptionStrategy.DISABLED,
    ),
)
```

The flush happens once the playback's echo-decay drain has elapsed — about two
seconds after the last audio is sent — and it logs at INFO so a session shows
plainly that nothing was dropped:

```
Flushing 1 queued speech segment(s) for <session> (playback finished)
```

Two consequences to design around:

- The caller's interjection is answered a few seconds late by construction.
  That delay is the point of the strategy, not a defect.
- The backlog is bounded. A caller who talks continuously through a very long
  answer will have their earliest segments dropped, with a warning, rather
  than growing the queue without limit.

Use it for announcements, legal disclaimers and any turn that must be heard in
full — while still hearing what the caller said during it.

## Backchannel Detection

The SEMANTIC strategy requires a `BackchannelDetector` to classify short utterances:

```python
from __future__ import annotations

from roomkit.voice.pipeline.backchannel import (
    BackchannelContext,
    BackchannelDecision,
    BackchannelDetector,
)


class KeywordBackchannelDetector(BackchannelDetector):
    """Simple keyword-based backchannel detector."""

    BACKCHANNELS = {"uh-huh", "yeah", "ok", "mhm", "right", "sure", "hmm", "yes"}

    @property
    def name(self) -> str:
        return "keyword_backchannel"

    def classify(self, context: BackchannelContext) -> BackchannelDecision:
        text = (context.transcript or "").strip().lower()
        is_bc = text in self.BACKCHANNELS and context.speech_duration_ms < 1000
        return BackchannelDecision(
            is_backchannel=is_bc,
            confidence=0.9 if is_bc else 0.1,
            label="acknowledgement" if is_bc else None,
        )
```

### BackchannelContext

The detector receives rich context for classification:

| Field | Type | Description |
|-------|------|-------------|
| `transcript` | `str \| None` | Transcribed text of the utterance |
| `speech_duration_ms` | `float` | Duration of the utterance |
| `audio_bytes` | `bytes \| None` | Raw audio for acoustic classifiers |
| `bot_speech_progress` | `float` | How far into the bot's utterance (0.0-1.0) |
| `session_id` | `str` | Voice session ID |
| `metadata` | `dict` | Additional context |

## Pipeline Integration

Interruption config can also be set via `AudioPipelineConfig`:

```python
from __future__ import annotations

from roomkit.channels import VoiceChannel
from roomkit.voice.interruption import InterruptionConfig, InterruptionStrategy
from roomkit.voice.pipeline import AudioPipelineConfig

pipeline = AudioPipelineConfig(
    vad=vad,
    denoiser=denoiser,
    backchannel_detector=my_detector,
    interruption=InterruptionConfig(
        strategy=InterruptionStrategy.SEMANTIC,
        min_speech_ms=300,
    ),
)

voice = VoiceChannel(
    "voice",
    stt=stt, tts=tts,
    backend=backend,
    pipeline=pipeline,
)
```

!!! note "Configuration Priority"
    1. Explicit `interruption` parameter on VoiceChannel (highest)
    2. `pipeline.interruption` from AudioPipelineConfig
    3. Legacy `enable_barge_in` + `barge_in_threshold_ms` (fallback)

## Hook Integration

Three hooks fire during interruption events:

!!! note "Where the speech is detected does not change who decides"

    Speech during playback can be noticed by the pipeline's VAD, by the
    continuous-STT energy check, or by a transport that does its own detection
    and calls `on_barge_in`. All three routes consult the same
    `InterruptionConfig`: a backend with native detection cannot override
    `DISABLED`, and it is subject to `allow_during_first_ms` like every other
    route. A transport reports *detected speech*, not a decision.

### ON_BARGE_IN

Fires when the user successfully interrupts the AI:

```python
from __future__ import annotations

from roomkit import HookTrigger, HookExecution

@kit.hook(HookTrigger.ON_BARGE_IN, execution=HookExecution.ASYNC)
async def on_barge_in(event, ctx):
    # event is a BargeInEvent
    print(f"User interrupted at {event.audio_position_ms}ms")
    print(f"AI was saying: {event.interrupted_text}")
```

### ON_TTS_CANCELLED

Fires when TTS playback is cancelled for any reason:

```python
@kit.hook(HookTrigger.ON_TTS_CANCELLED, execution=HookExecution.ASYNC)
async def on_tts_cancelled(event, ctx):
    # event is a TTSCancelledEvent
    # event.reason: "barge_in" | "explicit" | "disconnect" | "error"
    if event.reason == "barge_in":
        print(f"TTS stopped at {event.audio_position_ms}ms due to barge-in")
```

### ON_BACKCHANNEL

Fires when the SEMANTIC strategy detects a backchannel (not a real interruption):

```python
@kit.hook(HookTrigger.ON_BACKCHANNEL, execution=HookExecution.ASYNC)
async def on_backchannel(event, ctx):
    # event is a BackchannelEvent
    print(f"Backchannel: '{event.text}' (confidence={event.confidence})")
```

## Grace Period: allow_during_first_ms

The `allow_during_first_ms` field prevents interruptions during the first N milliseconds of playback. This is useful for:

- **Echo prevention**: When AEC isn't perfect, the first few hundred ms of playback can echo back and trigger false interruptions
- **Critical content**: Ensure the AI's opening words are always heard

```python
InterruptionConfig(
    strategy=InterruptionStrategy.IMMEDIATE,
    allow_during_first_ms=500,  # Don't interrupt during first 500ms
)
```

On `RealtimeVoiceChannel`, a provider such as OpenAI may own the VAD and cancel
its response before RoomKit receives the speech-start callback. When the
transport reports actual playback, RoomKit therefore enforces this grace period
at the provider-input boundary: AEC, AGC, denoising, and recording still receive
the real microphone signal, while the provider receives equal-duration PCM
silence. Once the interval expires, normal audio and barge-in resume. Transports
without playback callbacks cannot apply this physical-playback guard.

## Legacy Compatibility

The older `enable_barge_in` and `barge_in_threshold_ms` parameters still work and map to `InterruptionConfig` internally:

```python
# Legacy (still works):
VoiceChannel("voice", enable_barge_in=True, barge_in_threshold_ms=300)
# Equivalent to:
# InterruptionConfig(strategy=IMMEDIATE, allow_during_first_ms=300)

# Legacy disabled:
VoiceChannel("voice", enable_barge_in=False)
# Equivalent to:
# InterruptionConfig(strategy=DISABLED)
```

!!! warning
    Prefer the explicit `interruption` parameter over legacy fields for new code.

## Testing

Use `MockBackchannelDetector` for deterministic tests:

```python
from __future__ import annotations

from roomkit.voice.interruption import InterruptionConfig, InterruptionHandler, InterruptionStrategy
from roomkit.voice.pipeline.backchannel import BackchannelDecision
from roomkit.voice.pipeline.backchannel.mock import MockBackchannelDetector

# Test CONFIRMED strategy
config = InterruptionConfig(strategy=InterruptionStrategy.CONFIRMED, min_speech_ms=300)
handler = InterruptionHandler(config)
decision = handler.evaluate(playback_position_ms=500, speech_duration_ms=400)
assert decision.should_interrupt

# Test SEMANTIC with backchannel
detector = MockBackchannelDetector(
    decisions=[BackchannelDecision(is_backchannel=True, confidence=0.95)]
)
config = InterruptionConfig(
    strategy=InterruptionStrategy.SEMANTIC,
    backchannel_detector=detector,
)
handler = InterruptionHandler(config)
decision = handler.evaluate(playback_position_ms=500, speech_duration_ms=300, speech_text="uh-huh")
assert not decision.should_interrupt
assert decision.is_backchannel
```
