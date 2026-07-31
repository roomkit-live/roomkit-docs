# Buzz (Nostr) Channel

RoomKit talks to [Block's **Buzz**](https://github.com/block/buzz) — a
Nostr-based team workspace — through a **Nostr identity** (a keypair) that holds
a persistent, authenticated connection to the community's **relay**. Like
Discord, it is wired as a **source + provider pair** sharing one connection, via
the [`buzzkit`](https://pypi.org/project/buzzkit/) client:

- [`BuzzRelaySource`][roomkit.sources.buzz.BuzzRelaySource] owns the
  `buzzkit.BuzzClient`, authenticates (NIP-42), subscribes to a channel, and
  emits messages into RoomKit.
- [`BuzzProvider`][roomkit.providers.buzz.relay.BuzzProvider] reuses that client
  to publish replies over the relay's HTTP bridge.

A RoomKit **Room** maps to a Buzz **channel**; the agent's Nostr keypair is a
first-class, cryptographically-signed member of the community.

## Install

```bash
pip install roomkit[buzz]
```

`buzzkit` provides the Nostr bindings (Schnorr signing, event building) and the
async relay client.

## Join the community

Hosted Buzz communities are **closed relays**: the agent's key must be a member
before it can read or write. The membership-gate-exempt path is an **invite**:

1. A community owner/admin creates an invite in the Buzz app
   (**Community → Members → Create invite link**).
2. Claim it once for your agent key with `buzzkit`:

   ```python
   from buzzkit import BuzzClient, generate_keypair

   nsec, npub, _ = generate_keypair()   # your agent identity — store nsec securely
   client = BuzzClient("wss://your-community.communities.buzz.xyz", nsec)
   await client.claim_invite("https://your-community.communities.buzz.xyz/invite/<code>")
   ```

Then copy the target channel's UUID from the Buzz app.

## Wire it up

```python
from roomkit import BuzzChannel, RoomKit
from roomkit.providers.buzz import BuzzConfig, BuzzProvider
from roomkit.sources.buzz import BuzzRelaySource

config = BuzzConfig(
    relay_url="wss://your-community.communities.buzz.xyz",
    private_key="nsec1...",                     # the agent's Nostr secret
)
source = BuzzRelaySource(config, "buzz-main", relay_channel_id="<channel-uuid>")
provider = BuzzProvider(source)                 # reuses the source's client

kit = RoomKit()
kit.register_channel(BuzzChannel("buzz-main", provider=provider))
await kit.create_room(room_id="buzz-room")
await kit.attach_channel(
    "buzz-room", "buzz-main", metadata={"buzz_channel_id": "<channel-uuid>"}
)
await kit.attach_source("buzz-main", source)    # connects + subscribes
```

Register one channel per Buzz channel you want to bridge, and bind each to its
room. The recipient key `buzz_channel_id` resolves the target Buzz channel UUID
at delivery time.

## Inbound messages

Each Nostr event is parsed by
[`parse_buzz_event`][roomkit.sources.buzz.parse_buzz_event] into a RoomKit
`InboundMessage`:

- **Text** (kind-9 stream message) → `TextContent`.
- The sender's Nostr pubkey becomes `sender_id` (resolved to a participant).
- The Nostr event id becomes `external_id` and `idempotency_key`.
- **Threads** (NIP-10): the thread ROOT event id becomes `thread_id` (flat
  two-level model, like Slack's `thread_ts`), and the immediate parent lands in
  `metadata["nostr_reply_to"]`.

The agent's own events are always dropped (no echo loop;
`BuzzConfig.ignore_own=True`). Metadata includes `nostr_event_id`, `nostr_kind`,
and `buzz_channel_id`.

### Reactions

Reactions live **outside the message pipeline** — the same contract as the
Discord and WhatsApp channels. Pass an `on_event` callback to the source to
receive them (this widens the subscription to kinds 9 + 7 + 5):

```python
def on_relay_event(data: dict) -> None:
    # {"action": "add", "emoji": "🔥", "user_id": <pubkey>,
    #  "target_event_id": ..., "reaction_event_id": ..., "channel_id": ...}
    # or {"action": "remove", "user_id": ..., "reaction_event_id": ..., ...}
    ...

source = BuzzRelaySource(config, "buzz-main",
                         relay_channel_id=CHANNEL, on_event=on_relay_event)
```

A kind-5 deletion is surfaced as `action: "remove"` with the retracted
reaction's event id (Buzz retracts reactions by deleting the reaction event, so
the emoji/target are not on the wire). The agent's own reactions are dropped
under the same `ignore_own` policy as messages.

## Outbound messages

`provider.send(event, to=channel_uuid)` publishes `RoomEvent.content` as a Buzz
channel message (kind 9) over the relay's HTTP bridge, signed with the agent's
key. It returns a `ProviderResult` carrying the Nostr `event_id`. Because sends
use the HTTP bridge, they succeed even while the inbound WebSocket is
reconnecting.

To reply **in a thread**, set `channel_data.thread_id` to the thread-root
Nostr event id (the `thread_id` the inbound parser produced, or the message's
own `external_id` to start a thread under it) — the provider passes it as a
NIP-10 `reply_to`. To react, call
`provider.send_reaction(target_event_id, emoji)`; retract with
`provider.remove_reaction(reaction_event_id)` (the id a successful
`send_reaction` returned).

## Voice: huddles

Buzz **huddles** are live voice calls: when someone starts one in a channel,
the relay creates an **ephemeral channel** and announces it on the parent
channel (Nostr kind **48100**, with the huddle's id in
`content.ephemeral_channel_id`). Audio is **48 kHz mono Opus, one frame per
20 ms**, over the relay's `/huddle/{id}/audio` WebSocket. An agent that is a
member of the parent channel is admitted to its huddles automatically.

Two pieces bridge huddles to RoomKit's realtime voice stack:

- [`BuzzHuddleBackend`][roomkit.voice.backends.buzz_huddle.BuzzHuddleBackend]
  — the `VoiceBackend` that carries huddle audio for a
  `RealtimeVoiceChannel`.
- [`BuzzHuddleWatcher`][roomkit.voice.backends.buzz_huddle.BuzzHuddleWatcher]
  — the announcement→call lifecycle: watches the parent channel for
  kind-48100 events (through a `BuzzRelaySource` with `auto_restart`, so
  relay reconnection is handled), dials each huddle, bridges it, and rejoins
  if the relay drops the call mid-huddle.

```python
from roomkit import RealtimeVoiceChannel, RoomKit
from roomkit.providers.buzz import BuzzConfig
from roomkit.providers.gemini.realtime import GeminiLiveProvider
from roomkit.voice.backends.buzz_huddle import BuzzHuddleBackend, BuzzHuddleWatcher

kit = RoomKit()
voice = RealtimeVoiceChannel(
    "buzz-voice",
    provider=GeminiLiveProvider(api_key="..."),
    transport=BuzzHuddleBackend(),
)
kit.register_channel(voice)
await kit.create_room(room_id="huddles")
await kit.attach_channel("huddles", "buzz-voice")

watcher = BuzzHuddleWatcher(
    kit,
    voice_channel=voice,
    config=BuzzConfig(relay_url="wss://...", private_key="nsec1..."),
    parent_channel_id="<parent-channel-uuid>",
    room_id="huddles",
)
await watcher.start()          # or: await watcher.bridge("<huddle-uuid>")
```

### Buzz-specific behavior to know

- **The relay keeps a huddle alive while *any* member is connected — the
  agent included.** An agent that never hangs up leaves a zombie huddle
  behind. The backend therefore ends the session itself when the last remote
  peer leaves (`end_when_alone`, default on), with a grace period
  (`empty_huddle_grace`, 90 s) for huddles nobody has joined yet.
- **`session.metadata["buzz_end_reason"]`** tells you why a call ended:
  `"alone"` (call over) or `"connection_lost"` (relay dropped the socket).
  The watcher uses it to decide between rejoining and moving on.
- **The backend resamples internally** between the huddle's fixed 48 kHz and
  the provider's rates (defaults match Gemini Live: 16 kHz in / 24 kHz out).
  Do **not** set `transport_sample_rate` on the channel, or audio is
  resampled twice at the wrong rates.
- **Silence is streamed toward the provider** while the huddle is quiet
  (`silence_fill`, default on): huddle senders go silent between utterances
  (Opus DTX), but a realtime provider's server VAD needs to *hear* the
  post-speech silence to close the user's turn.
- **Outbound timing is owned by RoomKit's pacer.** The watcher creates
  huddle clients with `paced=False`; the backend paces frames with prebuffer
  and jitter headroom (same `OutboundAudioPacer` as the SIP backend). If you
  hand-build a `HuddleClient`, pass `paced=False` too.
- **A rejoin starts a fresh provider session** — the model does not remember
  the conversation from before the connection loss.
- **One call at a time**: announcements that arrive while a call is bridged
  are ignored.

For custom announcement handling (e.g. filtering which huddles to join),
subscribe your own source with `kinds=[KIND_HUDDLE_STARTED]` and
[`huddle_announcement_parser`][roomkit.sources.buzz.huddle_announcement_parser],
and call `watcher.bridge(huddle_id)` from your own hook.

## Capabilities & limits

`BuzzChannel` advertises text, with threading and reactions. Max message length
is 65536 characters (the relay's content limit).

Lifecycle details: a relay closing with code **1012** (graceful restart) is
reconnected quietly at the initial backoff — replayed events are deduped by
id. Presence is re-announced every 30 s (safe for both the 180 s and the older
90 s relay TTLs). `BuzzConfig.leave_on_stop` opts into a NIP-29 leave (kind
9022) when the source stops — leave it off for private channels, where an
admin granted the membership and self-join cannot get it back.

Out of scope for now: rich content and media (imeta). Each source subscribes
to a single relay channel.

## Runnable examples

See [`examples/buzz_bot.py`](https://github.com/roomkit/roomkit/blob/main/examples/buzz_bot.py)
for an end-to-end echo bot:

```bash
BUZZ_RELAY_URL=wss://your-community.communities.buzz.xyz \
BUZZ_NSEC=nsec1... \
BUZZ_CHANNEL_ID=<channel-uuid> \
uv run python examples/buzz_bot.py
```

And [`examples/buzz_voice_agent.py`](https://github.com/roomkit/roomkit/blob/main/examples/buzz_voice_agent.py)
for a speech-to-speech huddle agent (Gemini Live):

```bash
GOOGLE_API_KEY=... \
BUZZ_RELAY_URL=wss://your-community.communities.buzz.xyz \
BUZZ_NSEC=nsec1... \
BUZZ_CHANNEL_ID=<channel-uuid> \
uv run python examples/buzz_voice_agent.py
```
