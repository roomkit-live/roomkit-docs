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

The agent's own events are always dropped (no echo loop;
`BuzzConfig.ignore_own=True`). Metadata includes `nostr_event_id`, `nostr_kind`,
and `buzz_channel_id`.

## Outbound messages

`provider.send(event, to=channel_uuid)` publishes `RoomEvent.content` as a Buzz
channel message (kind 9) over the relay's HTTP bridge, signed with the agent's
key. It returns a `ProviderResult` carrying the Nostr `event_id`. Because sends
use the HTTP bridge, they succeed even while the inbound WebSocket is
reconnecting.

## Capabilities & limits

`BuzzChannel` advertises text, with threading and reactions. Max message length
is 65536 characters (the relay's content limit).

Out of scope for now: rich content, media (imeta), reaction dispatch, and voice
"huddles". Each source subscribes to a single relay channel.

## Runnable example

See [`examples/buzz_bot.py`](https://github.com/roomkit/roomkit/blob/main/examples/buzz_bot.py)
for an end-to-end echo bot:

```bash
BUZZ_RELAY_URL=wss://your-community.communities.buzz.xyz \
BUZZ_NSEC=nsec1... \
BUZZ_CHANNEL_ID=<channel-uuid> \
uv run python examples/buzz_bot.py
```
