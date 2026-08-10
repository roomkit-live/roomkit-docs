# Shared Microphone Capture

A `VoiceBackend` takes the capture device when a session starts and releases it
when that session ends. That is the right lifetime for a call, and the wrong
one for anything that has to listen *before* a session exists.

Consider a wake word. The detector holds the microphone, recognises the trigger
phrase, and must now hand the device over so the session can take it. The
reacquisition costs hundreds of milliseconds, and it lands precisely while the
person is still speaking — so the request is clipped and they have to repeat
it. No application buffer repairs this: a closed device captured nothing.

`AudioCaptureSource` moves device ownership out of the session. The detector
subscribes, the session subscribes too, and a **mark** taken when speech began
lets the session replay what was said before it existed.

Normative reference: RFC Section 12.12.

## The shape

```python
from roomkit.voice.capture import LocalMicSource

mic = LocalMicSource(sample_rate=24000, backlog_seconds=10)
mic.start()                                    # the device opens once

detector = mic.subscribe(enqueue, name="wakeword")   # no session in sight
```

The source retains recent audio in a bounded ring. `mark()` names a position in
it; `subscribe(since=mark)` replays from there before delivering anything live:

```python
# … your VAD reports SPEECH_START:
mark = mic.mark()

# … the utterance ends and the trigger matched:
await channel.start_session(
    room_id, participant_id, connection=None,
    metadata={"capture_since": mark},
)
```

That is the whole integration. `LocalAudioBackend` reads `capture_since` from
the session metadata, subscribes with it, and the replayed frames travel the
ordinary inbound path — landing in the realtime channel's pre-connect buffer
and flushing in order once the provider handshake completes. There is no new
control point on the inbound path and no direct call into the provider.

```python
from roomkit.voice.backends.local import LocalAudioBackend

transport = LocalAudioBackend(source=mic, output_sample_rate=24000)
```

With `source` set, the backend never opens a device of its own. Everything that
is genuinely per-session stays with it: mute, gating, half-duplex suppression
and echo cancellation.

## The subscriber contract

Fan-out is **synchronous on the capture thread**. That is what keeps echo
cancellation correct — capture and reference must stay in step, which they
cannot do across a queue hop. It also means:

> A subscriber must not perform unbounded work inside the callback. Enqueue the
> frame and return. The source guarantees no isolation between subscribers: one
> slow subscriber degrades capture for all of them.

The bar is lower than it sounds. A block is typically 20 ms, and a wake-word
subscriber running an ONNX encoder spends about that long per segment — enough
on its own to starve the device. So:

```python
def enqueue(frame):
    loop.call_soon_threadsafe(queue.put_nowait, frame)   # and nothing else

mic.subscribe(enqueue, name="wakeword")
```

The source times each callback and warns when one runs long, naming the
subscriber:

```
WARNING roomkit.voice.capture: Capture subscriber wakeword took 21.4ms on a
20ms block (47 slow callbacks so far); a subscriber must enqueue the frame and
return
```

Without that line, the only symptom is crackling audio with no named cause.

## Lifetime

`start()` and `stop()` are the only things that acquire and release the device.
Dropping to zero subscribers does **not** stop capture, and gaining one does not
start it. Ending a session, or closing the backend, therefore leaves the source
running — its lifecycle belongs to whoever created it.

This is what makes it safe for a detector to detach for the duration of a call
and reattach afterwards, which is the recommended pattern. Two reasons:

- Running the detector during a call burns CPU on audio nobody will act on.
- Source frames are **pre-AEC**. Echo cancellation is per-session — it needs the
  reference signal of what that session is playing — so it is applied
  downstream, in the backend, after fan-out. A detector left attached during a
  call hears the assistant's own voice unattenuated, and will trigger on it.

Resampling is likewise the subscriber's concern: a source has one format, and a
consumer that needs another converts it.

## Marks, and why not seconds

Addressing the backlog by duration — "replay the last 2.5 seconds" — forces the
caller to guess. Too short clips the request; too long replays the tail of the
previous conversation into the model. A mark placed where the speech actually
began has neither failure.

A mark whose position has already been evicted is **stale**. The source replays
what remains rather than failing, because raising at the moment a session opens
would discard the very utterance the backlog exists to preserve. The
degradation is reported, never silent:

```python
subscription = mic.subscribe(handler, since=mark)
subscription.truncated       # True when the mark had aged out
subscription.replayed_bytes  # what actually made it through
```

## The wake word is replayed too

Where the trigger phrase and the request share one utterance — "hey jarvis what
are you up to" — the backlog contains the trigger phrase as well. There is no
boundary to split them on, and inventing one would need word-level timestamps
the detector does not have. Providers handle the address form well. Treat it as
intended rather than filtering it.

## Detecting the wake word

Detection is the application's job, not RoomKit's — the source is agnostic
about what decides. One caveat is worth recording, because it costs a day to
rediscover.

**Transcribing the segment and searching the text for the trigger does not
work.** Measured on Whisper: constrained to French, foreign trigger words are
Frenchified — "hey jarvis" comes out "et j'arvisse", "ok kit" becomes "ok
quitte", a homophone that cannot be told apart from the word for *leave*.
Unconstrained, the model is sequence-to-sequence and interprets the segment
globally, picking one language for it: "hey jarvis tu fais quoi de beau" comes
back as "And I'll see what you do." — the trigger phrase is simply gone from
the text.

What works is an **acoustic fingerprint**, with no transcription at all: run
the VAD segment through the Whisper *encoder* alone (the tiny int8 encoder is
12 MB; the 97 MB decoder is never loaded), compare the embedding against
enrolled prototypes with cosine DTW, and threshold on a calibrated score.
Measured over 11 positives and 8 negatives, leave-one-out: 82% recall at 100%
precision, roughly 20 ms per segment, independent of language and spelling.
Log-mel features must use the Slaney scale — HTK puts Whisper out of
distribution and it returns noise.

## Testing

`MockCaptureSource` drives the whole path with no device:

```python
from roomkit.voice.capture import MockCaptureSource

source = MockCaptureSource()
source.start()

mark = source.mark()
phrase = source.feed_blocks(6)          # spoken before any session exists

await channel.start_session(room_id, "user", None,
                            metadata={"capture_since": mark})
# the provider received every block of `phrase`, in order
```

## Example

`examples/shared_mic_capture.py` runs the full path against Gemini Live. Its
trigger is deliberately trivial — the first speech segment opens a session — so
that it demonstrates the capture primitive without dragging in a detection
stack. Say a whole sentence in one breath: the assistant answers the question
instead of asking you to repeat it.
