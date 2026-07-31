# Conference (SFU Orchestration)

RoomKit can join a meeting it does not host. `ConferenceChannel` attaches a
room to a multi-party conference whose media plane an external SFU (Selective
Forwarding Unit) owns — LiveKit today, anything that can implement
`ConferenceBackend` tomorrow. The defining decision, and the first thing to
understand before anything else in this guide makes sense: **RoomKit is never
in the media path between humans.** The SFU routes their audio and video
directly; RoomKit orchestrates the conference and joins it as one more
participant — a bot — to provide the intelligence: transcription attributed to
whoever spoke, AI voice, recording, and cross-channel integration. RFC §12.10
is the normative reference (conformance Level 3).

## Quick start

Everything below runs against the mock backend — no SFU, no credentials:

```python
from roomkit import MockConferenceBackend, RoomKit
from roomkit.channels.conference import ConferenceChannel
from roomkit.voice.stt.mock import MockSTTProvider

kit = RoomKit()
conference = ConferenceChannel("conf", backend=MockConferenceBackend(), stt=MockSTTProvider())
kit.register_channel(conference)
await kit.create_room("standup")
await kit.attach_channel("standup", "conf")

# The credential a human's client uses to connect to the SFU directly.
await kit.ensure_participant("standup", "conf", "alice", display_name="Alice")
access = await conference.mint_access("standup", "alice")
```

For a real SFU, swap the backend:

```python
from roomkit import LiveKitConferenceBackend, LiveKitConfig

backend = LiveKitConferenceBackend(
    LiveKitConfig(url="wss://my-project.livekit.cloud")  # key/secret from env
)
```

Install with: `pip install roomkit[livekit]`

## How it works

```
 alice ──┐                              ┌── RoomKit bot participant
         │        ┌──────────┐          │     ├─ subscribes audio tracks
 bob ────┼───────►│   SFU    │◄─────────┘     │    └─ lane per track → VAD → STT
         │        │ (LiveKit)│                ├─ publishes one AI voice track
 carol ──┘        └──────────┘                └─ observes joins, mutes, quality
                       ▲
              media stays here — RoomKit
              never routes a human's packets
```

Six design principles carry the whole section (RFC §12.10.1), and every
behavior described later falls out of one of them:

1. **RoomKit orchestrates; the SFU transports.** Per-packet routing, codecs,
   simulcast, bandwidth estimation — the SFU's job. RoomKit MUST NOT sit in
   the media path between human participants.
2. **The conference is the room.** One conference maps 1:1 to a Room.
   Conference participants are Room participants, transcriptions are
   RoomEvents, and hooks, permissions and cross-channel broadcast apply with
   no conference-specific exceptions.
3. **Human clients connect directly to the SFU.** RoomKit mints access
   credentials and observes lifecycle through backend events; it never
   proxies client signaling or media. Client SDKs are provider-specific and
   out of scope.
4. **RoomKit joins as a bot participant.** One server-side connection per
   conference: subscribed tracks feed the audio pipeline, and the AI's voice
   is published as a single bot track the SFU distributes.
5. **Tracks are the unit of media identity.** Every stream carries its
   publisher's identity, so speaker attribution comes from track identity —
   which is why the conference pipeline has no diarization stage, and (since
   clients cancel their own echo and the SFU never mixes) no server-side AEC
   either.
6. **The backend boundary is decoded frames and opaque credentials.** The
   `ConferenceBackend` delivers decoded PCM and mints credentials the
   framework never inspects. SDP, ICE, codecs and bitrates appear in no
   framework interface.

## Constructor parameters

```python
ConferenceChannel("conf", backend=backend, stt=stt, tts=tts)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `backend` | `ConferenceBackend` | required | The SFU transport |
| `stt` | `STTProvider \| None` | `None` | Transcription; without any consumer the bot never subscribes audio |
| `tts` | `TTSProvider \| None` | `None` | The bot's voice; without it the bot cannot speak |
| `pipeline` | `AudioPipelineConfig \| None` | `None` | Explicit pipeline; the default builds a 16 kHz mono contract + energy VAD |
| `interruption` | `ConferenceInterruptionConfig \| None` | immediate, any | Who may barge in, and how eagerly |
| `recording` | `ConferenceRecordingConfig \| None` | `None` | Per-track recording (needs `recorder=`) |
| `recorder` | `MediaRecorder \| None` | `None` | Where recordings go — see [Room Media Recorder](room-media-recorder.md#conference-recording) |
| `bot_identity` | `str` | `"roomkit"` | The bot's participant identity in the SFU |
| `bot_grants` | `ConferenceGrants \| None` | derived | Defaults to `for_bot(speaks=tts is not None, listens=...)` |
| `default_grants` | `ConferenceGrants \| None` | all-publish | What `mint_access()` grants humans |
| `e2ee` | `bool` | `False` | End-to-end encrypted room; refused together with `stt` — the bot could not read the media it is asked to transcribe |
| `close_room_on_detach` | `bool` | `False` | Also `close_room()` on the SFU when detaching |
| `speak_text_events` | `bool` | `False` | Speak non-AI text events too — off because a meeting is not a place to read other channels' traffic aloud |
| `close_providers` | `bool` | `True` | Close STT/TTS/pipeline when the channel closes |
| `max_queued_frames` | `int` | `100` | Per-lane backpressure bound (~2 s of audio) |
| `identity_address_keys` | `Sequence[str] \| None` | `CONFERENCE_ADDRESS_KEYS` | Which provider attributes can found an identity |
| `identity_trusts_unasserted_metadata` | `bool` | `False` | Trust participant-supplied attributes — see [Identity Resolution](identity-resolution.md) |

## The lazy join

The bot does not join when the channel attaches. It joins when there is a
first reason to be there, and every trigger is something the framework can see
without the backend's help (RFC §12.10.4 step 1):

- **A credential is minted.** `mint_access()` is the framework's own advance
  notice that a human is about to connect — by the time the tab opens, the
  bot is on the participant list. No backend callback could do this: presence
  is only observable through a connection, and this is the *first* join.
- **The attach probes occupancy.** An attach may land over a conference
  already under way — a channel restarted mid-meeting re-attaches above
  participants an earlier life admitted, with no mint left to wait for. The
  attach calls `list_participants()` off its own path, and anyone in there
  who is not the bot starts the join. Ops note: the INFO log
  `found N participant(s) already in <room> at attach` names this trigger,
  and the join's catch-up then rebuilds the roster. An empty conference stays
  unjoined — an idle room costs one control-plane call, not a connection.
- **An event needs delivering** (the AI has something to say), or **a
  participant arrives** on a conference the bot already left.

### Pure transport mode

A reason is only a reason if the channel can do something with the
connection. The join exists for the intelligence — subscribed tracks feed
stt and recording through it, tts publishes on it — so a channel constructed
with none of the three never joins at all: the mint and the arrival start no
join, and the occupancy probe is not made, the join being the only
consequence a probe can have (RFC §12.10.4 step 1).

What remains is exactly what a purely human meeting needs from RoomKit:
`ensure_room()` creates the conference, `mint_access()` admits participants,
arrivals a backend reports are still recorded on the roster, and `info()`
answers truthfully — `bot_present: false`, nothing active. What is given up
is the event bridge: the bot's connection is how presence becomes observable
(RFC §12.10.3), so against a real SFU a pure-transport conference produces
no `participant_joined`/`left`, active-speaker or quality callbacks — the
framework's view of the meeting is its mints. If your obligations require
attendance tracking, configure a consumer or track presence through your own
client surface; a bot kept in the meeting only to watch it is the silent
observer RFC §17.7 exists to surface, and RoomKit does not offer that as a
mode.

Configure any one of `stt=`, `tts=` or `recording=` and every trigger above
is restored unchanged.

## Lanes: how transcription scales to N speakers

Each subscribed AUDIO track gets a **lane** of its own — a bounded queue and a
task, keyed by track id. The backend's frame callback returns immediately;
a slow recognizer on one participant's track never delays another's frames.

Within a lane the stages run in pipeline order (RFC §12.10.4):

```
frames ──► [Resampler] ──► [AGC] ──► [Denoiser] ──► VAD ──► STT
```

Two stages you know from the [voice pipeline](audio-pipeline-stages.md) are
deliberately absent. AEC: there is no server-side echo path — clients cancel
their own echo, and the SFU never mixes audio. Diarization: the track's
identity already attributes the speech.

**Segmentation is the point.** Audio arrives as 20 ms frames; nothing is sent
to the STT while a participant speaks. The lane accumulates, and when the VAD
has seen enough consecutive silence the whole utterance leaves as one block:
one utterance becomes **one** RoomEvent, attributed to the track's publisher —
not one event per frame. The [mock quickstart example][example-mock] prints
the frame count next to the transcription count to make the ratio visible.

Configuration follows the "default that works" rule: without a `pipeline=`
argument the channel builds a 16 kHz mono contract with an
`EnergyVADProvider`. Pass an `AudioPipelineConfig` to choose your own VAD —
but an explicit config *without* one is refused at construction when an STT
is configured: a lane cannot segment without a VAD, and failing at
construction beats transcribing nothing. For real meetings prefer a neural
VAD ([sherpa-onnx](sherpa-onnx.md) TEN-VAD): an energy threshold loses
softly-spoken words and never closes an utterance over background noise.

**Backpressure is bounded and counted.** A lane's queue holds
`max_queued_frames` (default 100 — about two seconds). At saturation the
*oldest* frame is dropped and counted on `lane.dropped_frames`: the policy is
to stay near live rather than accumulate delay, and the count is how you know
it happened. The trade is deliberate — a transcript that lags the meeting by
growing seconds is worth less than one with a hole it can report.

## The three read surfaces

A management UI asks three different questions, and RoomKit answers each from
the authority that actually knows — deliberately not through one aggregate
call:

| Surface | Authority | What it knows |
|---------|-----------|---------------|
| `backend.list_participants()` | The SFU | **Media presence**: who is connected right now, since when, with which tracks (kind, muted) and which metadata the SFU asserts — a dial-in carries its number here |
| `kit.store.list_participants()` | RoomKit's roster | **The room's records**: role, status, identification, resolved identity, display name — survives the meeting |
| `conference.info()` | The channel | **The RFC §17.7 disclosure**: bot present, collection permitted, STT *active* (as distinct from configured), recording active, dropped frames — at the moment you ask |

The SFU forgets a participant on disconnect; the roster remembers them; and
neither can answer "is anything listening right now", which is `info()`'s
job. A UI composes the three. The [LiveKit example][example-livekit] dumps
all three side by side.

Two facts about the bot's own presence belong here. First, where there is
intelligence it is not optional: media is only observable through a
connection, so transcription, recording and AI voice all require the bot in
the meeting — there is no "STT without the participant". The reverse holds
too: with no stt, tts or recording configured, the bot never joins at all
and `info()` says so — see [Pure transport mode](#pure-transport-mode).
Second, *visible* is a grant, not a law: a
`ConferenceGrants` with `hidden=True` as `bot_grants` keeps the bot off the
meeting's participant lists where the SFU supports it (LiveKit does) — but
`info()` reports `bot_hidden` either way. Whether a silent notetaker is
acceptable is jurisdiction and policy, the integrator's to decide; RFC §17.7
makes sure they can always answer for what is in the room.

## Events

Everything a meeting does reaches you through hooks (see the trigger table in
RFC §9) and framework events. The lifecycle triggers are ASYNC — observers
that cannot block:

| Trigger | Payload data | Notes |
|---------|--------------|-------|
| `ON_CONFERENCE_PARTICIPANT_JOINED` / `_LEFT` | `participant_id` | The SFU's arrivals and departures |
| `ON_PARTICIPANT_UPDATED` | participant | Fired by `rename_member()` |
| `ON_CONFERENCE_TRACK_PUBLISHED` / `_UNPUBLISHED` | `track_id`, `participant_id`, `kind` | What a client actually sent |
| `ON_CONFERENCE_TRACK_MUTED` / `_UNMUTED` | `track_id`, `participant_id`, `kind` | A muted VIDEO track is how most clients say "camera off" — microphone *and* camera indicators both read from this pair |
| `ON_SCREEN_SHARE_STARTED` / `_STOPPED` | `participant_id` | A published/unpublished `SCREEN_SHARE` track |
| `ON_SPEECH_START` / `ON_SPEECH_END` | `participant_id`, `track_id` | The VAD's utterance boundaries, per lane, in real time — no STT round-trip |
| `ON_ACTIVE_SPEAKER_CHANGED` | `participant_id` | The SFU's dominant speaker. It cannot say that *nobody* is speaking — the end of a turn is `ON_SPEECH_END`'s to announce |
| `ON_CONNECTION_QUALITY_CHANGED` | `participant_id`, `quality` | As the SFU sees it — the bot included, which is the notetaker's own health signal |
| `ON_BARGE_IN` | `ConferenceBargeIn` | Who interrupted the bot, and mid-what |
| `ON_RECORDING_STARTED` / `_STOPPED` | `ConferenceRecordingStarted/Stopped` | Consent point and result, per track |

Alongside the hooks, `kit.on(...)` framework events mark the conference
itself: `conference_started` (with `bot_session_id`), `conference_ended`
(with `duration_ms`), `conference_participant_joined` / `_left`. Register
listeners *before* the attach — in a resume, the join's catch-up redelivers
the participants already in the meeting within its first seconds, and a
listener registered after the attach misses them.

### ON_TRANSCRIPTION is synchronous, and that is a feature

One trigger stands apart. `ON_TRANSCRIPTION` fires **synchronously**, with a
`ConferenceTranscription` payload (`room_id`, `track_id`, `participant_id`,
`text`), *before* the text becomes a RoomEvent:

```python
from roomkit import ConferenceTranscription, HookResult
from roomkit.models.enums import HookTrigger

@kit.hook(HookTrigger.ON_TRANSCRIPTION)
async def redact(payload: ConferenceTranscription, ctx) -> HookResult:
    clean = strip_card_numbers(payload.text)
    if clean != payload.text:
        return HookResult.modify(payload.model_copy(update={"text": clean}))
    return HookResult.allow()
```

A hook may allow, rewrite (`HookResult.modify` with an updated payload) or
suppress (`HookResult.block`) — which is where redaction lives, and why the
trigger cannot be async: a redaction that ran after the broadcast would be
too late. It **fails closed**: a hook that raises blocks the text, because
carrying on with the original would publish exactly what the hook existed to
suppress. Two practical notes: a sync hook must *return* a `HookResult` —
returning `None` is treated as no verdict and, on this trigger, the text is
dropped — and a slow hook holds up that lane's transcript, so keep it fast.

## Names

`display_name` makes a round trip the integrator never has to manage. The
name on the room's roster rides `mint_access()` into the minted token; the
SFU's clients render it; the SFU reports it back on `list_participants()` and
the join's catch-up; and a roster record *without* a name takes the reported
one — never overwriting one the integrator set. The practical consequence:
names survive a restart with no persistent store, because the SFU still knows
them and the catch-up writes them back.

To rename someone mid-meeting, `kit.rename_member(room_id, participant_id,
display_name)` — presentation-only, changes what a member is *called* and
never who they are, announced on `PARTICIPANT_UPDATED` /
`ON_PARTICIPANT_UPDATED`. Calling `add_member()` on an already-ACTIVE member
is deliberately a no-op — re-admitting is not renaming.

## Interruption

Who may talk over the bot is policy, not physics:

```python
from roomkit import ConferenceInterruptionConfig, ConferenceInterruptionScope

ConferenceChannel(
    "conf", backend=backend, stt=stt, tts=tts,
    interruption=ConferenceInterruptionConfig(
        scope=ConferenceInterruptionScope.ALLOWLIST,  # or ANY (default), NONE
        allowlist=["alice"],
    ),
)
```

When a barge-in lands, `ON_BARGE_IN` receives a `ConferenceBargeIn` naming
the interrupting participant, the interrupted text and how far into the audio
it happened — and the transport's queued audio is discarded
(`ConferenceBackend.stop_playback()`) instead of playing out to the end of
its buffer, so the interruption is as immediate as the transport allows.

The strategies are the voice channel's
([`InterruptionStrategy`](voice-interruption.md)) with one exception:
`SEMANTIC` is refused at construction. Backchannel classification needs the
utterance's *text*, which exists only once the utterance has ended — too late
to interrupt anything. Refusing at construction beats a policy that silently
cannot fire.

## Access and security

The security model in one sentence: **RoomKit is the gate for issuing
tokens, not for the connection; between the two stands the TTL; and eviction
is reactive, never preventive.**

`mint_access()` is where policy is enforced. It validates *before* it mints:
the participant must be on the room's roster (`ensure_participant` /
`add_member` first), a banned participant is refused
(`ParticipantNotAdmittedError` — and bans stick: no SFU event lifts one), an
unknown one raises `ParticipantNotFoundError`, a room the channel is not
attached to (or is leaving) refuses too. What it grants is
`default_grants` — a `ConferenceGrants` deciding publish/subscribe/moderate —
and the credential it returns is opaque to the framework: `ConferenceAccess`
carries the SFU's URL and token, which the client uses directly.

After the mint, the credential is live until its TTL expires
(`LiveKitConfig.access_ttl`, default 15 minutes) — revoking a ban during that
window does not un-issue tokens already out. What the framework can do is
evict a participant already connected (`remove_participant()`), which is
reactive by design: there is no pre-connection hook, because the connection
happens at the SFU, not through RoomKit. Size `access_ttl` to how long a
"click the link" window should stay open, not to the meeting's length.

One more SFU behavior worth knowing: a second connection with the same
identity **evicts the first** (LiveKit reports `DUPLICATE_IDENTITY`). A
leaked token does not let someone lurk alongside its owner — but it does let
them take the owner's seat; the TTL is what bounds that exposure.

## Resilience

A bot session can end without anyone calling `leave()`: the connection drops,
the SFU evicts the bot, the backend's event bridge overflows. A backend that
observes this reports it through `on_bot_session_ended`, and the channel
takes the session off its books, finalizes its recordings, announces
`conference_ended`, and re-joins under a bounded, backed-off supervisor
(`REJOIN_DELAYS_S` — 0.5 s to 8 s over five attempts) for as long as the room
stays attached and collectable. Past the attempts, the lazy join remains the
safety net: the next mint, delivery or arrival brings the bot back. What
happened *inside* the outage window is genuinely lost, and reported as a
discontinuity rather than as observed-and-empty — §17.7 implementations
should treat it that way.

The honest caveat: a backend that *cannot* observe the loss inherits the
failure mode knowingly — no report, no supervisor, only the lazy join.

Relatedly: a room holds at most one conference. Attaching a second conference
channel is refused with `ConferenceAlreadyAttachedError`, and the reservation
outlives the binding — a room whose previous conference channel still has a
session in the meeting, or a teardown running, keeps refusing. The refusal is
retryable, never a wait: try again once the previous teardown has finished.

## Writing a ConferenceBackend

`ConferenceBackend` is the ABC an SFU integration implements. The boundary is
strict in both directions: the backend delivers **decoded PCM audio** (frames
that declare their format — the lane resamples, the backend must not) and
mints **opaque credentials**; SDP, ICE, codecs and simulcast never cross it.
The other rule worth internalizing before writing one: the backend is a pure
transport. No VAD, no segmentation, no speech smarts — RoomKit owns those,
and a backend that pre-segmented audio would break the separation the ABC
exists to draw (it is why the LiveKit backend uses `livekit.rtc` +
`livekit.api` and deliberately not `livekit-agents`).

| Group | Methods |
|-------|---------|
| Identity | `name`, `capabilities` (a `ConferenceCapability` flag set) |
| Room lifecycle | `ensure_room()`, `close_room()`, `close()` |
| Admission | `mint_access()`, `list_participants()`, `remove_participant()` |
| Moderation | `mute_track()`, `unmute_track()` (behind `REMOTE_UNMUTE`) |
| Bot session | `join_as_bot()`, `leave()` |
| Media | `subscribe_track()`, `unsubscribe_track()`, `publish_audio()`, `stop_playback()`, `publish_video()` (behind `VIDEO_PUBLISH`) |

Events flow the other way through registration methods the ABC provides
(`on_participant_joined`, `on_track_published`, `on_track_audio`,
`on_active_speaker_changed`, `on_connection_quality`, `on_bot_session_ended`,
…) — implement the SFU-event-to-callback bridge, and declare in
`capabilities` what is actually wired rather than what the SFU sells.

Two references pay off here. `LiveKitConferenceBackend`
(`roomkit/conference/livekit.py`) is the production template — connection
management, an ordered event bridge with bounded memory, and per-track frame
pumps. `MockConferenceBackend` is both the test double and the contract
demonstration: it implements the whole ABC with `simulate_*` methods for
every event, plus fault injection (`fail()`, `delay()`) to make failure paths
reachable — see [Testing Patterns](testing-patterns.md#mock-conference-backend).

## Recording and identity, briefly

Both have their own guides. With `recorder=` and
`recording=ConferenceRecordingConfig()`, every subscribed track is recorded
*separately*, attributed to its publisher, the bot's own audio included;
`ON_RECORDING_STARTED` is a consent point (a handler that refuses captures
nothing), and audio-only — there is no video egress. The details live in
[Room Media Recorder](room-media-recorder.md#conference-recording).

A conference is also where participants the framework did not admit arrive —
dial-ins, out-of-band admissions. Who gets identified, from which metadata,
and why the SFU's own assertions are the only trustworthy ones is
[Identity Resolution](identity-resolution.md)'s conference section.

## Limits, honestly

Current boundaries, each tracked rather than papered over:

- **One worker per conference.** The one-room-one-conference reservation's
  authority is a single RoomKit instance; running the same room's conference
  from several workers over a shared store is not yet a contract.
- **Roster ghosts after a hard discontinuity.** With a persistent store, a
  participant who left *while no RoomKit was alive to see it* can linger
  ACTIVE on the roster until the next catch-up corrects it.
- **Recording consent and encryption at rest** are the integrator's to
  provide today; the framework's part (the `ON_RECORDING_STARTED` consent
  point) exists, the rest is tracked separately.
- **Audio only.** The channel declares audio capability alone: no vision on
  conference video tracks yet, no bot-published video (nothing produces it
  until an avatar does), no video egress recording.

## Relationship to audio bridging

The in-process [AudioBridge](audio-bridge.md) also connects N parties — by
mixing their audio inside RoomKit, which puts RoomKit in the media path and
caps how far it scales. That is the boundary the conference exists to cross
(RFC §12.10.10): when the parties are humans in a meeting, the SFU carries
the media and RoomKit steps out of the path. Bridge for a handful of calls
RoomKit itself terminates; conference for meetings.

## API Reference

See the [Conference API Reference](../api/conference.md) for auto-generated
class documentation.

## Examples

- [`examples/conference_quickstart.py`][example-mock] — the whole
  arrangement on the mock backend: mint, lanes, attributed transcriptions,
  the frame-to-utterance ratio, and why silence produces nothing.
  Deterministic; runs with no SFU and no credentials.
- [`examples/conference_livekit.py`][example-livekit] — the same against a
  real LiveKit SFU: two browser tabs, one bot, speech attribution by ear,
  interruption, teardown, and a resume mode that proves the occupancy probe.
- [`examples/conference_fault_injection.py`][example-faults] — testing
  against a backend that fails, lags and varies its audio formats.

[example-mock]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_quickstart.py
[example-livekit]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_livekit.py
[example-faults]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_fault_injection.py
