# Audio Prompts (Playing WAV Files)

`VoiceChannel.play()` sends a pre-recorded WAV straight to the transport — no TTS synthesis, no LLM round-trip. It is the telephony primitive behind welcome messages, IVR menus, hold music, compliance disclosures and "all our agents are busy" announcements.

Its sibling `say()` goes through TTS. Pick by what you have:

| You have | Method | Path |
|---|---|---|
| Text | `voice.say(session, "…")` | TTS provider → outbound pipeline → backend |
| A recorded file | `voice.play(session, wav)` | WAV → outbound pipeline → backend |

Both are proactive: they speak without an inbound message and without the AI being involved.

---

## Playing a file

```python
await voice.play(session, "prompts/welcome.wav")
await voice.play(session, wav_bytes, text="[welcome message]")
```

`audio` accepts a path (`str` or `pathlib.Path`) or raw WAV bytes already in memory — useful when prompts live in object storage or a TTS cache.

`text` is a display transcript only: it is forwarded to the transport via `send_transcription()` for the UI. It is not stored in the room timeline.

`play()` needs no TTS provider — a channel wired with nothing but a backend can still play prompts. Only `say()` requires TTS.

**The WAV must be 16-bit PCM, mono, uncompressed.** Anything else raises `ValueError` before a byte is sent. Convert once, up front:

```bash
ffmpeg -i prompt.mp3 -ac 1 -ar 8000 -acodec pcm_s16le prompt.wav
```

`await play()` returns when the audio has drained. On SIP the pacer clocks packets out at real time, so the await lasts roughly the length of the file. Run it as a task if the caller should hear the prompt while your code keeps working.

---

## Sample rates and the pipeline contract

The file's sample rate does **not** have to match the codec. When the channel has an `AudioPipelineConfig` with a `contract`, the outbound resampler converts every prompt to the transport rate — a resampler is created automatically as soon as a contract is present:

```python
from roomkit.voice.pipeline import AudioFormat, AudioPipelineConfig, AudioPipelineContract

CODEC_RATE = 8000   # G.711 (PCMU/PCMA); use 16000 for G.722

pipeline = AudioPipelineConfig(
    vad=vad,
    contract=AudioPipelineContract(
        transport_inbound_format=AudioFormat(sample_rate=CODEC_RATE),
        transport_outbound_format=AudioFormat(sample_rate=CODEC_RATE),
        internal_format=AudioFormat(sample_rate=16000),
    ),
)
voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend, pipeline=pipeline)
```

With that contract, a 22 050 Hz prompt reaches the SIP leg as 8 kHz audio (verifiable in `examples/voice_sip_play_prompt.py`, which prints the byte counts on both sides).

!!! warning "No pipeline, no resampling"
    Without a pipeline — or without a contract — `play()` hands the WAV to the backend untouched, and the SIP backend clocks those bytes out at the negotiated codec rate whatever the file says. A 44.1 kHz prompt sent to an 8 kHz leg is stretched to five times its length, an octave and a half too low. Either configure the contract or pre-convert the file to the codec rate.

---

## Getting the session

`play()` needs the `VoiceSession`. The safe point is `ON_SESSION_STARTED`, which fires when the audio path is live in both directions:

```python
from roomkit.models.session_event import SessionStartedEvent

@kit.hook(HookTrigger.ON_SESSION_STARTED, execution=HookExecution.ASYNC)
async def on_session_started(event: SessionStartedEvent, ctx: object) -> None:
    if event.session is not None:
        await voice.play(event.session, "prompts/welcome.wav")
```

For playback later in the call — an IVR menu after a DTMF digit, hold music before a transfer — keep the session from that hook (or from `@backend.on_call`) in a dict keyed by room.

---

## Keeping the AI quiet

A prompt the AI talks over, or answers, is worse than no prompt. Two mutes cover the two cases, and they compose:

| Call | Effect | Use when |
|---|---|---|
| `await kit.mute(room_id, "ai")` | The AI's brain still runs (memory ingest, context build) but its voice is dropped: a streamed reply is closed before the provider round-trip — no tokens spent — and a non-streamed one is generated then discarded | Hold music: the caller should still be heard and transcribed |
| `await kit.mute(room_id, "voice")` | Inbound audio is dropped before the VAD: no STT, no barge-in, nothing reaches the AI | Announcements and disclosures that must not be interrupted |

Muting never blocks `play()` itself — playback writes to the backend directly instead of going through the broadcast pipeline.

A context manager keeps the two calls balanced:

```python
import contextlib

@contextlib.asynccontextmanager
async def ai_silenced(kit, room_id, *, deafen=True):
    await kit.mute(room_id, "ai")
    if deafen:
        await kit.mute(room_id, "voice")
    try:
        yield
    finally:
        if deafen:
            await kit.unmute(room_id, "voice")
        await kit.unmute(room_id, "ai")


async def play_prompt(kit, voice, session, room_id, wav, *, deafen=True):
    async with ai_silenced(kit, room_id, deafen=deafen):
        await voice.interrupt(session)   # cut any TTS already in flight
        await voice.play(session, wav, text="[prompt]")
```

!!! note "Muting silences the voice, not the brain"
    This is the RFC rule for muted channels (§7). A muted AI channel still ingests the conversation, so when you unmute it, it already knows what happened. Only the outbound speech was suppressed.

---

## Interrupting playback

`interrupt()` stops a prompt mid-file:

```python
stopped = await voice.interrupt(session)   # True if something was playing
```

It pops the playback state, calls `cancel_audio()` on the backend (SIP flushes the pacer and the send buffer) and resets AEC.

Two behaviours to know about:

- **Barge-in cuts prompts too.** With `enable_barge_in=True` (the default), a caller who speaks during a prompt interrupts it, exactly as with TTS. For an announcement that must be heard in full, mute the voice channel for its duration (`deafen=True` above) — dropped frames never reach the VAD, so barge-in cannot fire.
- **A prompt stays "live" for ~2 s after the audio drains.** That window lets continuous STT discard echo of your own prompt instead of transcribing it as caller speech. `interrupt()` ends the window immediately if you need the next turn to start clean.

---

## What `play()` does not do

- **No `BEFORE_TTS` / `AFTER_TTS` hooks.** Those belong to the synthesis path; `play()` bypasses it. Gate prompts in your own code.
- **No timeline event.** The room's history does not record that a prompt was played, so the AI's context will not mention it. If the AI needs to know, write it yourself:

```python
from roomkit import TextContent

await voice.play(session, "prompts/recording_notice.wav")
await kit.send_event(
    room_id,
    "voice",
    TextContent(body="[played: recording notice]"),
    addressed_to=[],   # store it, do not ask any agent to answer it
)
```

- **No transcription of the prompt.** `text=` is a UI label, not an event.

---

## Realtime (speech-to-speech) channels

`RealtimeVoiceChannel` has no `play()` — it owns the audio path end to end with the provider. To hold it quiet, mute the provider's audio at the transport boundary:

```python
await kit.mute_output(room_id, "realtime")     # provider audio stops reaching the transport
...
await kit.unmute_output(room_id, "realtime")
```

Prompt audio then has to be written to the backend yourself, **already at the codec rate** (the realtime channel's own resampler is not in this path):

```python
await backend.send_audio(session, pcm_bytes_at_codec_rate)
```

---

## Full example

`examples/voice_sip_play_prompt.py` runs both wirings:

```bash
uv run python examples/voice_sip_play_prompt.py               # mock transport, no infrastructure
SIP_MODE=1 uv run python examples/voice_sip_play_prompt.py    # real SIP trunk, needs roomkit[sip]
```

The mock run walks through four states and prints what reached each stage: the welcome prompt (with the resampled byte count), a caller talking over a deafened prompt (nothing transcribed), a turn with only the AI muted (transcribed and stored, no speech), and a normal turn (the AI answers).

## See also

- [Voice Greeting](voice-greeting.md) — greeting patterns, including `say()` and agent `auto_greet`
- [SIP Voice Backend](sip-backend.md) — trunk setup, codecs, call routing
- [Voice Interruption & Barge-In](voice-interruption.md) — interruption strategies
- [Audio Resampler](resampler.md) — resampler providers and the pipeline contract
