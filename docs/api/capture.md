# Audio Capture Sources

A capture source owns an audio input device independently of any voice session,
so that a consumer which must listen *before* a session exists — a wake word, a
level meter — no longer has to hand the device over at the moment the person
starts speaking.

Normative reference: RFC Section 12.12. Narrative guide:
[Shared Microphone Capture](../guides/shared-mic-capture.md).

## Quick start

```python
from roomkit.voice.capture import LocalMicSource
from roomkit.voice.backends.local import LocalAudioBackend

mic = LocalMicSource(sample_rate=24000, backlog_seconds=10)
mic.start()

# Listen with no session in sight. Enqueue only — never block the callback.
detector = mic.subscribe(enqueue, name="wakeword")

# The backend becomes a subscriber rather than the device's owner.
transport = LocalAudioBackend(source=mic)

mark = mic.mark()                       # at SPEECH_START
await channel.start_session(            # once the trigger matched
    room_id, participant_id, connection=None,
    metadata={"capture_since": mark},
)
```

Install the device implementation with `pip install roomkit[local-audio]`.

See the [full example](https://github.com/roomkit-live/roomkit/blob/main/examples/shared_mic_capture.py)
for a complete runnable script.

## Source ABC

::: roomkit.voice.capture.base.AudioCaptureSource

## Marks and subscriptions

::: roomkit.voice.capture.base.CaptureMark

::: roomkit.voice.capture.base.CaptureSubscription

## Implementations

::: roomkit.voice.capture.local.LocalMicSource

::: roomkit.voice.capture.mock.MockCaptureSource

## The subscriber contract

Fan-out is synchronous on the capture thread — that is what keeps echo
cancellation's capture/reference timing in step. A subscriber must not perform
unbounded work in the callback: enqueue the frame and return. The source
guarantees no isolation between subscribers, times each callback, and warns by
name when one runs long.

Frames are emitted raw, **pre-AEC**: echo cancellation is per-session and
applied downstream in the backend. A subscriber left attached during a call
hears the far end unattenuated.
