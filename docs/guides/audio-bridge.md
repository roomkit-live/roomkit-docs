# Audio Bridging

Audio bridging enables direct session-to-session audio forwarding for
human-to-human voice calls.  Audio flows between participants without
the STT/TTS roundtrip, preserving natural voice quality and minimizing
latency.

Bridging mixes audio *inside* RoomKit, which puts the framework in the media
path and caps how far it scales. For multi-party meetings at SFU scale —
where RoomKit steps out of the media path entirely — see
[Conference (SFU Orchestration)](conference.md) (RFC §12.10.10).

## Quick Start

Enable bridging by passing `bridge=True` to `VoiceChannel`:

```python
from roomkit import RoomKit, VoiceChannel
from roomkit.voice.backends.sip import SIPVoiceBackend

backend = SIPVoiceBackend(
    local_sip_addr=("0.0.0.0", 5060),
    local_rtp_ip="0.0.0.0",
)

voice = VoiceChannel("voice", backend=backend, bridge=True)

kit = RoomKit()
kit.register_channel(voice)
```

When two participants join the same room and are bound to the voice
channel, audio from each is forwarded to the other automatically.

## How It Works

```
Session A mic → Backend → Inbound Pipeline → AudioBridge.forward()
                                                  │
                                                  ├─→ Outbound Pipeline → Session B speaker
                                                  └─→ VAD/STT (optional, parallel)
```

1. **Inbound audio** from each session passes through the full inbound
   pipeline (resampler, AEC, AGC, denoiser, VAD).
2. **AudioBridge** receives the processed frame and forwards it to all
   other sessions in the same room.
3. **Outbound pipeline** processes each forwarded frame per target
   (recorder tap, AEC reference feeding, resampler) before it reaches
   the transport.
4. **STT path** (if configured) runs in parallel -- bridging and
   transcription are independent.

## Configuration

### AudioBridgeConfig

```python
from roomkit.voice.bridge import AudioBridgeConfig

config = AudioBridgeConfig(
    max_participants=10,         # Max sessions per room (default: 10)
    mixing_strategy="forward",   # "forward" (2-party) or "mix" (N-party)
)

voice = VoiceChannel("voice", backend=backend, bridge=config)
```

| Parameter | Default | Description |
|---|---|---|
| `enabled` | `True` | Whether bridging is active. Set `False` to pause forwarding. |
| `max_participants` | `10` | Maximum bridged sessions per room. Raises `RuntimeError` if exceeded. |
| `mixing_strategy` | `"forward"` | `"forward"` sends frames directly. `"mix"` does N-party additive mixing. |
| `mixer` | Auto | Mixer provider for PCM mixing. Auto-detects NumPy (fast) or falls back to pure Python. |

## N-Party Mixing

For conference calls with 3+ participants, use `mixing_strategy="mix"`:

```python
config = AudioBridgeConfig(mixing_strategy="mix")
voice = VoiceChannel("voice", backend=backend, bridge=config)
```

Each participant hears a mix of all **other** participants' audio
(excluding their own).  The mixing algorithm:

- **2 sources**: summed sample-by-sample with int16 clipping
- **3+ sources**: averaged (divided by N) with int16 clamping

All arithmetic uses int32 accumulation to avoid overflow during
intermediate steps.

### MixerProvider

The mixer is pluggable via the `MixerProvider` ABC:

```python
from roomkit.voice.bridge import AudioBridgeConfig
from roomkit.voice.pipeline.mixer.numpy import NumpyMixerProvider
from roomkit.voice.pipeline.mixer.python import PythonMixerProvider

# Explicit NumPy mixer (~20x faster than pure Python)
config = AudioBridgeConfig(
    mixing_strategy="mix",
    mixer=NumpyMixerProvider(),
)

# Pure Python fallback (no dependencies)
config = AudioBridgeConfig(
    mixing_strategy="mix",
    mixer=PythonMixerProvider(),
)

# Auto-detect (default): NumPy if installed, else Python
config = AudioBridgeConfig(mixing_strategy="mix")
```

## Cross-Rate Resampling

When participants have different native sample rates (e.g., SIP at 8kHz
and WebRTC at 48kHz), the bridge automatically resamples audio to match
each target's rate:

```
SIP (8kHz) ──► Bridge ──► resample to 48kHz ──► WebRTC participant
WebRTC (48kHz) ──► Bridge ──► resample to 8kHz ──► SIP participant
```

In mix mode, all source frames are resampled to the target's rate
**before** mixing to ensure temporal alignment.  Resampling uses a
cached `LinearResamplerProvider` singleton for efficiency.

When a `frame_processor` is set (i.e., the outbound pipeline handles
resampling), the bridge skips its own resampling to avoid double
resampling.

## Bridge + STT (Transcription)

Bridge mode and STT run in parallel.  This is useful for compliance
recording, live transcription display, or AI monitoring:

```python
voice = VoiceChannel(
    "voice",
    backend=backend,
    bridge=True,
    stt=deepgram_provider,
)
```

Audio frames pass through both the bridge (forwarding to other
participants) and the VAD/STT pipeline (transcription) simultaneously.

## Bridge + TTS (AI Moderator)

When both bridge and TTS are active, the AI can speak into a bridged
call via `say()`.  TTS audio and bridge audio coexist -- neither blocks
the other:

```python
voice = VoiceChannel(
    "voice",
    backend=backend,
    bridge=True,
    stt=deepgram_provider,
    tts=elevenlabs_provider,
)

# AI can speak into the bridged call
await voice.say(session, "This call may be recorded for quality purposes.")
```

!!! note "Echo suppression during TTS"
    After TTS playback completes, there is a 2-second echo decay window
    during which speech detection is suppressed.  Schedule AI
    interjections before or well after participant speech segments.

## Bridge + Interruption

When a participant speaks while TTS is playing (barge-in), the
interruption handler cancels TTS but does **not** stop bridge
forwarding.  This ensures:

- AI speech is interrupted (TTS cancelled)
- Human-to-human bridge audio continues uninterrupted
- The participant's speech is still forwarded to other bridged sessions

## Frame Filtering

Use `set_bridge_filter()` to inspect or modify audio frames before they
are forwarded.  The filter runs synchronously in the audio callback
thread and must complete quickly (< 1ms):

```python
# Mute a specific participant
def mute_filter(session, frame):
    if session.id == muted_session_id:
        return None  # drop frame (mute)
    return frame

voice.set_bridge_filter(mute_filter)

# Remove the filter
voice.set_bridge_filter(None)
```

The filter receives `(source_session, AudioFrame)` and returns:

- The frame (unchanged or modified) to forward it
- `None` to drop the frame (mute the source for that frame)

## Hooks

### BEFORE_BRIDGE_AUDIO

The `BEFORE_BRIDGE_AUDIO` hook fires for each audio frame before it is
forwarded to other sessions.  It supports `HookResult.block()` to drop
individual frames:

```python
@kit.hook(HookTrigger.BEFORE_BRIDGE_AUDIO, execution=HookExecution.SYNC)
async def monitor_bridge(event, ctx):
    # event.session — source session
    # event.frame — the AudioFrame about to be forwarded
    # event.room_id — the room where bridging is active

    if should_mute(event.session):
        return HookResult.block(reason="participant_muted")
    return HookResult.allow()
```

!!! tip "Performance: fast path"
    When no `BEFORE_BRIDGE_AUDIO` hooks are registered, the bridge
    forwards frames directly in the audio thread with zero overhead.
    When hooks are registered, frames are routed through the event loop
    for hook evaluation.

For synchronous, ultra-low-latency frame filtering (< 1ms), use
`set_bridge_filter()` instead.

### Other Voice Hooks

These hooks fire normally during bridge mode:

- `ON_SPEECH_START` / `ON_SPEECH_END` -- VAD still runs on bridged audio
- `ON_TRANSCRIPTION` -- if STT is configured
- `ON_RECORDING_STARTED` / `ON_RECORDING_STOPPED` -- recorder captures both sides
- `ON_SESSION_STARTED` -- session lifecycle unchanged

## Session Lifecycle

Sessions are registered with the bridge when bound to the voice channel
and unregistered when unbound:

```python
# Join: binds session and adds to bridge
# Previously voice.bind_session() / voice.unbind_session()
await kit.join("room-1", "voice", session=session)

# Leave: unbinds session and removes from bridge
await kit.leave(session)
```

## Thread Safety

All bridge operations are thread-safe.  `AudioBridge.forward()` is
called from audio callback threads (the same context as
`on_audio_received`).  Internal state is protected by a
`threading.Lock`.

## Examples

### Mock Conference + AI Summary

`examples/voice_bridge_summary.py` — 3-party mock conference with
live transcription.  When all participants leave, an AI summarizes the
conversation.

```bash
uv run python examples/voice_bridge_summary.py
```

### SIP Bridge + Deepgram + Anthropic

`examples/voice_sip_bridge_summary.py` — Real SIP calls bridged with
Deepgram STT (multilingual auto-detect) and Claude for meeting summaries
in the same language as the speakers.

```bash
DEEPGRAM_API_KEY=... ANTHROPIC_API_KEY=... \
    uv run python examples/voice_sip_bridge_summary.py
```

### Bridge + AI Moderator

`examples/voice_bridge_with_ai.py` — Two participants bridged while an
AI moderator interjects via TTS.  Demonstrates bridge + STT + TTS
coexistence.

```bash
uv run python examples/voice_bridge_with_ai.py
```
