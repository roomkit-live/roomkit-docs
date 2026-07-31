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
is restored unchanged — and none of the three has to be configured at
construction: the next section is how a running channel gains and loses them.

## Hot-plugging intelligence

The configuration first need is read from is not fixed at construction (RFC
§12.10.4). Each kind of intelligence can be plugged into and unplugged from a
channel that is serving live conferences:

```python
# The meeting is running, purely human. The host asks for a notetaker:
await conference.plug_stt(stt)
await conference.plug_recording(ConferenceRecordingConfig(), recorder=recorder)

# Later, the host dismisses it:
await conference.unplug_stt()
await conference.unplug_recording()   # last need gone — the bot leaves
```

Six methods, one per kind and direction: `plug_stt(stt, pipeline=None)` /
`unplug_stt()`, `plug_tts(tts)` / `unplug_tts()`,
`plug_recording(config, recorder=...)` / `unplug_recording()`. All are
`async`, and their effects are in force when they return.

**Plugging a need is a first need.** In every attached room, an occupied
conference is joined before the plug returns — the attach's occupancy probe
is re-run, because the join is once more a consequence a probe can have —
and the tracks already published are subscribed retroactively: the meeting
is transcribed from the plug forward, not from the next publication. An
empty conference stays unjoined, and the ordinary lazy-join triggers stand
ready. A probe or join that fails does not fail the plug: the configuration
stands, and the next mint, delivery or arrival retries — the same discipline
as every other trigger.

**Unplugging the last need takes the bot out.** A session kept past the last
consumer and the last voice is the silent observer §17.7 refuses, so it
leaves, `conference_ended` is announced, and the channel stands down exactly
as a channel constructed pure-transport — same channel, same room, no
rebuild. An unplug that leaves other needs standing keeps the bot: unplug
the stt of a channel that still records and the lanes close while the
subscriptions — recording still consumes them — stay. An utterance in flight
when the voice is unplugged is ended the way a barge-in ends one
(`stop_playback` plus the terminal chunk): the conference is live, and the
turn genuinely ended.

**The grants follow.** Derived bot grants are computed from the
configuration in force at each join, and a plug that widens what a live
session must do — a voice needs `publish_audio`, a consumer needs
`subscribe` — brings the session's grants in line before the plug returns.
Where the backend declares `ConferenceCapability.BOT_GRANT_UPDATE` (LiveKit
does — `UpdateParticipant` is part of the server API), the update happens in
place: same session, same subscriptions, the event bridge intact. Where it
cannot, the channel re-joins — a `leave()` and a `join_as_bot()`, each
announced as the session event it is. A narrowing the backend cannot apply
in place is left standing: an unused privilege against a cut in the event
bridge is a trade settled for continuity. An explicit `bot_grants=` is never
rewritten by a plug or an unplug — the caller who set it took coverage on
themselves.

**Explicit grants are owned — and their owner can speak at runtime.** That
the plugs never rewrite an explicit `bot_grants=` makes it owned, not
frozen: `set_bot_grants()` is the owner's runtime voice (RFC §12.10.4).
Pass a new grant set to replace the explicit one — the coverage bargain
carries forward, so a set that does not cover the plugged needs is accepted
exactly as construction would accept it — or pass `None` to return the
channel to derivation. Unlike a plug's alignment, the change is an
instruction: it is applied to every live session in full, in place where
the backend can, and by the announced re-join where it cannot or where the
in-place update fails.

The flagship move is the observer who reveals itself:

```python
# Hidden from the first second — the event bridge without the listening:
conference = ConferenceChannel(
    "conf", backend=backend, stt=stt, bot_grants=ConferenceGrants.observer()
)
# Mid-meeting the host starts the notetaker, and the policy says: visible.
await kit.unmute(room_id, "conf")                        # collection opens
await conference.set_bot_grants(ConferenceGrants.for_bot(listens=True))
```

On LiveKit the reveal happens in place — `UpdateParticipant` removes
`hidden` and the SFU announces the bot to the clients already connected
(verified live): same session, same subscriptions, the event bridge intact.
Concealment is the asymmetric half: no SFU interface can *un-tell* clients
about a participant they were told of, so a visible→hidden change always
replaces the session — the announced leave is the retraction — whatever the
backend's capabilities. `info()["bot_grant_update_in_place"]` says
beforehand which price a change will carry, the per-room
`info()["rooms"][id]["bot_hidden"]` reports the status in force on the
session (§17.7), and every effective change on a connected session emits
the `conference_bot_grants_changed` framework event.

**Refusals and ownership.** A plug refuses exactly what construction
refuses — `stt` or `recording` on an E2EE conference, a recording mode with
no egress surface — and a slot already holding a provider is refused rather
than silently replaced: a swap is a teardown and a rebuild whatever single
verb offers it, so the observation gap belongs in the open — unplug, then
plug. Unplugging an empty slot is a no-op. An unplugged provider is closed
under the same `close_providers` rule that governs the channel's own
`close()` (the default `True` closes it; pass `False` to keep a shared
provider alive). And `info()` answers §17.7 with the configuration in
force, not the constructor's: a meeting that gained a transcriber mid-way
reads as transcribed from that moment, and one that lost it reads as clean.

See [`examples/conference_notetaker_on_demand.py`][example-notetaker] for
the whole round trip — transport-only to intelligent and back — driven by a
host's explicit request, which is exactly the consent gesture §17.7 wants
the decision tied to.

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

## The AI in the meeting

The flagship combination — a human speaks, the model understands, the bot
answers out loud — is not a feature you wire. It is principle 2 doing its
job: transcriptions are RoomEvents, cross-channel broadcast applies with no
conference-specific exceptions, so attaching an `AIChannel` to the room *is*
the STT → LLM → TTS loop:

```python
from roomkit.channels.ai import AIChannel
from roomkit.providers.anthropic import AnthropicAIProvider, AnthropicConfig

conference = ConferenceChannel("conf", backend=backend, stt=stt, tts=tts)
ai = AIChannel("ai", provider=AnthropicAIProvider(AnthropicConfig(api_key=...)),
               system_prompt="You are the meeting's voice assistant.")
kit.register_channel(conference)
kit.register_channel(ai)
await kit.attach_channel("standup", "conf")
await kit.attach_channel("standup", "ai")
# done — speak, and the bot answers on its own track
```

One channel rule makes the voicing automatic: a conference speaks only
events from an AI-typed source (`speak_text_events=True` widens that to
everything, and is off because a meeting is not a place to read other
channels' traffic aloud). The model's answer is such an event, so it is
synthesized and published on the bot track with zero configuration; any
other channel's text passes through silently.

A loop that answers a room it also speaks into needs two protections, and
both are already in place:

- **The media loop.** The bot's own TTS must not come back through a lane,
  be transcribed, and be answered again. Bot self-exclusion closes it: the
  channel never subscribes to a track its own session published
  (RFC §12.10.4).
- **The event loop.** The AI must not answer its own answer. `AIChannel`
  skips events it produced, and the framework bounds any AI-to-AI chain at
  `max_chain_depth` (default 5, RFC §8.3) — two AIs in one room converse
  finitely.

Between the answer and the audio sit the orchestration points. `BEFORE_TTS`
runs synchronously with the final text just before synthesis: it may rewrite
the bot's words or block them — a pending handoff is the classic reason to
hold the bot silent — and blocking keeps the answer out of the *audio*, not
out of the room's record. `AFTER_TTS` fires with what was actually said. And
who may cut the bot off mid-answer is policy, not accident — see
[Interruption](#interruption).

The deterministic version of all of this — including a handoff holding an
answer back, and both loop protections measured — is
[`examples/conference_ai_meeting.py`][example-ai]; export `ANTHROPIC_API_KEY`
before running [`examples/conference_livekit.py`][example-livekit] and the
same loop runs live: you talk, the bot talks back.

## Speech-to-speech: the realtime AI as a participant

The STT → LLM → TTS loop trades latency for structure. When the meeting wants
a sub-second agent with native turn-taking instead, one realtime provider
(Gemini Live, OpenAI Realtime, …) replaces all three stages
(RFC §12.10.12):

```python
from roomkit import ConferenceRealtimeConfig
from roomkit.providers.gemini import GeminiLiveProvider

conference = ConferenceChannel(
    "conf",
    backend=backend,
    realtime=ConferenceRealtimeConfig(
        provider=GeminiLiveProvider(api_key=...),
        system_prompt="You are the meeting's voice assistant.",
    ),
    stt=stt,  # optional, and recommended: see attribution below
)
```

The shape of the composition: every subscribed audio track is mixed N→1 —
additive, headroom-scaled, in 20 ms windows, with silence-only windows never
forwarded — and fed to **one provider session per conference**, established
lazily on the first mixed window and torn down with the bot's own session.
The provider's voice publishes on the bot track under the ordinary utterance
contract: floor, terminal `is_final`, one utterance at a time. To the SFU and
to a barge-in, a realtime response and a TTS answer are indistinguishable.

Three arbitrations to know about:

- **Attribution ends at the provider boundary.** The provider heard a mix,
  so its own transcription of the room names nobody — RoomKit discards it
  rather than guess. Configure `stt=` beside `realtime=` and the per-track
  lanes run in parallel with the mix, keeping the transcript attributed
  exactly as in the STT/TTS pattern. The provider's *assistant* finals are
  kept: they are the only record of what the AI said, and they land as room
  events attributed to the channel.
- **The per-lane VAD stays the barge-in sensor.** The provider brings its
  own turn-taking on the mix, but *who* may cut the bot off is the
  conference's [interruption policy](#interruption), and only track identity
  can name the interrupter. A barge-in that lands stops the chunk stream,
  discards what the SFU had queued (`stop_playback`), and tells the provider
  to cancel the response — best-effort, since not every provider can cancel
  (Gemini Live's `interrupt()` is a documented no-op; its own detection
  usually beats the framework to it anyway).
- **One voice per bot.** `tts=` and `realtime=` are mutually exclusive: both
  publish on the one bot track. The provider *is* the voice, so inbound text
  events that pass the speaking gate are injected into its conversation
  context (`inject_text`) instead of being synthesized over it.

The slot hot-plugs like every other need — `plug_realtime(config)` /
`unplug_realtime()`, with the occupancy probe re-run at the plug and the bot
retiring on the last unplug — and the lanes it shares with a recognizer
survive whichever of the two unplugs first. `info()` discloses
`realtime_configured`, `realtime_provider`, and per room `realtime_active`
(the session is actually connected) and `realtime_dropped_windows` (audio
the mix discarded to stay near-live).

The deterministic walkthrough — lazy session, attributed transcript beside
the mix, a barge-in reaching the provider as a cancellation — is
[`examples/conference_realtime_ai.py`][example-realtime].

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
  With `ANTHROPIC_API_KEY`, a live `AIChannel` answers the meeting out loud.
- [`examples/conference_ai_meeting.py`][example-ai] — the STT → LLM → TTS
  loop, deterministic on the mock backend: the AI answers a spoken question,
  `BEFORE_TTS` holds an answer back during a handoff, and both anti-loop
  protections are measured rather than asserted.
- [`examples/conference_realtime_ai.py`][example-realtime] — speech-to-speech
  N→1, deterministic on the mocks: the mixed meeting reaches one realtime
  session lazily, the provider's voice closes on `is_final`, and a barge-in
  stops the playback and cancels the response.
- [`examples/conference_notetaker_on_demand.py`][example-notetaker] — the
  hot-plug round trip: a purely human meeting gains its notetaker when the
  host asks (join, retroactive subscription, transcription from the plug
  forward) and loses it when dismissed (the bot leaves, recordings
  finalized).
- [`examples/conference_fault_injection.py`][example-faults] — testing
  against a backend that fails, lags and varies its audio formats.

[example-ai]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_ai_meeting.py
[example-realtime]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_realtime_ai.py
[example-mock]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_quickstart.py
[example-livekit]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_livekit.py
[example-notetaker]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_notetaker_on_demand.py
[example-faults]: https://github.com/roomkit-live/roomkit/blob/main/examples/conference_fault_injection.py
