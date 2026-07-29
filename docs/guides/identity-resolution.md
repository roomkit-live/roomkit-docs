# Identity Resolution

Identity resolution maps inbound message senders to known participants. It runs automatically in the inbound pipeline — after message parsing and before broadcast hooks.

## Quick Start

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.identity.base import Identity, IdentityResolver, IdentityResult
from roomkit.models.enums import IdentificationStatus
from roomkit.models.events import InboundMessage


class CRMResolver(IdentityResolver):
    """Resolve identity from a CRM database."""

    async def resolve(self, message: InboundMessage, context) -> IdentityResult:
        user = await crm_db.lookup_by_phone(message.sender_id)
        if user:
            return IdentityResult(
                status=IdentificationStatus.IDENTIFIED,
                identity=Identity(
                    id=user.id,
                    display_name=user.name,
                    email=user.email,
                    phone=message.sender_id,
                ),
            )
        return IdentityResult(status=IdentificationStatus.UNKNOWN)


kit = RoomKit(
    identity_resolver=CRMResolver(),
    identity_timeout=10.0,  # Timeout in seconds (default: 10)
)
```

## IdentificationStatus

The resolver returns one of 6 statuses:

| Status | Meaning | Participant Created | Message Blocked |
|--------|---------|--------------------|-----------------|
| `IDENTIFIED` | Known identity | Yes (identified) | No |
| `PENDING` | Awaiting resolution | Depends on hook | Depends on hook |
| `AMBIGUOUS` | Multiple candidates | Depends on hook | Depends on hook |
| `UNKNOWN` | No match found | Depends on hook | Depends on hook |
| `CHALLENGE_SENT` | Verification sent | No | Yes |
| `REJECTED` | Access denied | No | Yes |

## Identity Model

```python
from roomkit.identity.base import Identity

identity = Identity(
    id="user-123",
    display_name="Alice Smith",
    email="alice@example.com",
    phone="+1234567890",
    channel_addresses={
        "sms": ["+1234567890"],
        "email": ["alice@example.com"],
        "whatsapp": ["+1234567890"],
    },
    external_ids={"crm": "CRM-456", "stripe": "cus_abc"},
    metadata={"tier": "premium", "department": "engineering"},
)
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Unique identity ID (required) |
| `display_name` | `str \| None` | Human-readable name |
| `email` | `str \| None` | Email address |
| `phone` | `str \| None` | Phone number |
| `channel_addresses` | `dict[str, list[str]]` | Per-channel-type addresses |
| `external_ids` | `dict[str, str]` | External system IDs |
| `metadata` | `dict` | Arbitrary data |

## Identity Hooks

When the resolver returns `AMBIGUOUS`, `PENDING`, `UNKNOWN`, or `REJECTED`, hooks fire to let you override the decision.

### ON_IDENTITY_AMBIGUOUS

Fires when the resolver returns multiple candidates:

```python
from __future__ import annotations

from roomkit import HookTrigger, RoomKit
from roomkit.models.identity import IdentityHookResult

kit = RoomKit(identity_resolver=my_resolver)


@kit.identity_hook(HookTrigger.ON_IDENTITY_AMBIGUOUS)
async def on_ambiguous(event, context, id_result):
    # Pick the best candidate based on your logic
    best = id_result.candidates[0]
    return IdentityHookResult.resolved(best)
```

### ON_IDENTITY_UNKNOWN

Fires when the resolver finds no match:

```python
@kit.identity_hook(HookTrigger.ON_IDENTITY_UNKNOWN)
async def on_unknown(event, context, id_result):
    # Option 1: Allow as pending participant
    return IdentityHookResult.pending(display_name="Anonymous")

    # Option 2: Reject the message
    # return IdentityHookResult.reject(reason="Unknown sender blocked")
```

### ON_PARTICIPANT_IDENTIFIED

Fires after a participant is successfully identified (informational, cannot modify):

```python
@kit.hook(HookTrigger.ON_PARTICIPANT_IDENTIFIED)
async def on_identified(event, ctx):
    logger.info(f"Identified: {event.source.participant_id}")
```

## IdentityHookResult Factory Methods

| Method | Status Set | Effect |
|--------|-----------|--------|
| `IdentityHookResult.resolved(identity)` | `IDENTIFIED` | Create identified participant |
| `IdentityHookResult.pending(display_name)` | `PENDING` | Create pending participant |
| `IdentityHookResult.challenge(inject, message)` | `CHALLENGE_SENT` | Block message, send verification |
| `IdentityHookResult.reject(reason)` | `REJECTED` | Block message entirely |

## Challenge/Response Flow

Send a verification challenge and block the original message until the user identifies themselves:

```python
from __future__ import annotations

from roomkit import HookTrigger
from roomkit.models.identity import IdentityHookResult
from roomkit.models.events import InjectedEvent, TextContent


@kit.identity_hook(HookTrigger.ON_IDENTITY_UNKNOWN)
async def challenge_unknown(event, context, id_result):
    # Inject a verification request
    challenge = InjectedEvent(
        content=TextContent(body="Please reply with your account number to continue."),
        channel_id=event.source.channel_id,
    )
    return IdentityHookResult.challenge(
        inject=challenge,
        message="Verification challenge sent",
    )
```

When `CHALLENGE_SENT` is returned:

1. The original inbound message is **blocked** (not broadcast)
2. The injected event is delivered to the sender's channel
3. The sender's next message goes through identity resolution again

## Configuration

```python
from __future__ import annotations

from roomkit import RoomKit
from roomkit.models.enums import ChannelType

kit = RoomKit(
    identity_resolver=my_resolver,
    identity_channel_types={ChannelType.SMS, ChannelType.WHATSAPP},  # Only these channels
    identity_timeout=10.0,  # Seconds before timeout (default: 10)
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `identity_resolver` | `None` | The resolver instance. `None` disables resolution |
| `identity_channel_types` | `None` | Restrict to specific channel types. `None` = all. Applies to conference arrivals too — [below](#turning-it-off) |
| `identity_timeout` | `10.0` | Timeout in seconds. On timeout, status becomes `UNKNOWN` |

## Pipeline Position

```
Inbound Message
  → InboundRoomRouter.route()       # Find target room
  → Channel.handle_inbound()        # Parse → RoomEvent
  → IdentityResolver.resolve()      # <-- Identity resolution here
  → Identity hooks (if needed)
  → BEFORE_BROADCAST hooks
  → Store event
  → Broadcast
```

### When resolution is skipped

A resolver maps an *address* — a number, an email, a handle — to an `Identity`.
Two senders carry no such question, and the framework does not ask
(RFC §11.6):

| Case | What it means |
|------|---------------|
| The room has already identified the sender | The event's `participant_id` names a `Participant` of the room whose `identification` is `IDENTIFIED`. The answer is on the roster; `identity_id` already carries it |
| The channel names its own senders | The channel sets `sender_is_participant = True`, declaring its `sender_id` is a room `Participant.id` rather than an address. `ConferenceChannel` does — see below |

Skipping matters beyond the saved lookup. Re-resolving a sender the room already
identified returns `UNKNOWN` — no resolver matches a framework identifier — so
`ON_IDENTITY_UNKNOWN` fires again, and the standard refusal pattern:

```python
@kit.identity_hook(HookTrigger.ON_IDENTITY_UNKNOWN)
async def refuse(event, ctx, id_result):
    return IdentityHookResult.reject("unknown sender")
```

would block every message from a participant the framework itself identified.

`PENDING` and `AMBIGUOUS` participants are deliberately *not* skipped: a
participant the room has is not one it has identified, the resolver may still be
what settles it, and a hook may still want to challenge or refuse.

A channel that carries real addresses must leave `sender_is_participant` at its
default of `False` — declaring it wrongly stops those addresses ever being
resolved.

### Which channels name their own senders

The test is where the `sender_id` comes from. A channel that reads it off the
wire carries whatever the remote network put there. A channel RoomKit itself
names the sender of carries a `Participant.id`, and declares it:

| Channel | Declares | Why |
|---|---|---|
| `ConferenceChannel` | ✅ | An utterance carries the identity its track was published under — a `Participant.id`. The conference resolves once, when the participant arrives (below) |
| `CLIChannel` | ✅ | `run()` names the human at the keyboard; its `sender_id` defaults to `"user"` and is a `Participant.id`, not an address |
| `WebSocketChannel` | ❌ | Whatever the integrator puts on `sender_id` — an address in one deployment, an internal id in another. Excluding it is a configuration call: `identity_channel_types` |
| `VoiceChannel` | ❌ | `sender_id` is the `VoiceSession.participant_id` the backend filled in — a SIP session id for one call, a caller number for the next. Deployment-dependent |
| Transport channels (SMS, email, RCS, WhatsApp, chat…) | ❌ | The address *is* the sender: that is the question a resolver exists to answer |

The rule of thumb: declare it when RoomKit chose the value, leave it alone when
the network or the integrator did.

## Conferences: the participant the framework did not name

A conference participant that RoomKit admitted arrives already named — the id
passed to `mint_access()` comes back from the SFU. One it did not admit (a
SIP/PSTN dial-in, or an admission arranged out of band) arrives under the
backend's own opaque identity, `sip_15551234`, which no resolver can match. The
address that *can* be matched is in the provider's participant attributes, and
`ConferenceChannel` passes it to the resolver on its own.

Two things about this differ from the inbound pipeline above:

- **It runs when the participant arrives, not when it first speaks.** Someone
  can sit through a whole meeting without publishing a word; waiting for speech
  would leave them unidentified to every hook and roster read in the meantime.
  The arrival is also the *only* point at which a conference resolves: a
  transcription enters the inbound pipeline under the identity its track was
  published under — a `Participant.id`, not an address — so `ConferenceChannel`
  sets `sender_is_participant = True` and utterances skip resolution entirely
  (RFC §11.6). Speaking again asks nothing new, and no identity hook fires per
  sentence.
- **The result is linked to the participant, not substituted for it.** The Room
  `Participant.id` stays the backend's identity — that is what attributes
  transcription events, per-track recordings and the interruption allowlist —
  and the resolved Identity is carried on `identity_id`:

```python
participant = await kit.store.get_participant("room-1", "sip_15551234")
participant.id              # "sip_15551234"  — the backend's identity
participant.external_id     # "sip_15551234"
participant.identity_id     # "user-42"       — who it turned out to be
participant.identification  # IdentificationStatus.IDENTIFIED
```

So a caller dialling into a conference reaches the same `Identity` it would have
reached by texting the room, and nothing downstream has to change identifier
halfway through the meeting.

### Who put the address there

Which key carries the address is the second question. The first is who wrote the
value, because on most SFUs one attribute map carries two very different things:

| | |
|---|---|
| **Asserted** — `ConferenceParticipant.asserted_metadata` | Facts the SFU established: the number a SIP trunk reported, a claim in a token it authenticated, an attribute set through a server-side API. |
| **The rest** — everything else in `ConferenceParticipant.metadata` | What a participant's own client supplied when it joined. Surfaced, never vouched for. |

Only the first kind founds an identity. An attribute a client supplied is a claim
about itself: a caller writing its own `phone_number` and resolved on it reaches
whichever `Identity` that number belongs to — someone else's — and the
`Participant` then carries the victim's `identity_id` on the record every later
attribution reads.

```python
# The SFU asserts the trunk's number → identified.
await backend.simulate_participant_joined(
    "room-1", "sip_15551234", metadata={"sip.phoneNumber": "+15551234"}
)

# The participant writes it itself → the resolver is never called.
await backend.simulate_participant_joined(
    "room-1", "sip_9", client_metadata={"sip.phoneNumber": "+15551234"}
)
```

Provenance outranks specificity: an asserted address on a generic key beats an
unasserted one on the provider's own key, because an attacker chooses the key and
never the provenance.

A backend that cannot tell the two apart says so by leaving `asserted_metadata`
as `None`, and nothing is founded on what it surfaces. If your deployment has a
reason to trust it anyway — a closed client fleet, provenance you establish
elsewhere — say so explicitly:

```python
channel = ConferenceChannel("conf", backend=backend, identity_trusts_unasserted_metadata=True)
```

### Which attribute counts as an address

Among the asserted attributes, the channel reads a documented list of keys, most
specific first, and takes the first non-empty string it finds:

```python
from roomkit import CONFERENCE_ADDRESS_KEYS

CONFERENCE_ADDRESS_KEYS
# ("sip.phoneNumber", "phone_number", "phoneNumber",
#  "caller_id", "callerId", "from_number")
```

`sip.trunkPhoneNumber` — the number the caller *dialled* — is deliberately absent:
every dial-in reaches the same trunk, so reading it would identify all of them as
one person. So is `from`: it is a SIP header name generic enough that a value
found under it says nothing about where it came from. If your provider names the
caller's number differently, say so rather than forking:

```python
channel = ConferenceChannel(
    "conf",
    backend=backend,
    identity_address_keys=("x-caller-number", *CONFERENCE_ADDRESS_KEYS),
)
```

When no address is found, the participant stays `UNKNOWN`: the channel does not
fall back to resolving on the opaque identity.

### Where the provider's attributes end up

A conference is the one place where strangers write into a `Participant`'s
metadata, so what the provider attached lives under a key of its own, with its
provenance kept and its size bounded — never merged flat over the fields your
own code put there:

```python
from roomkit import CONFERENCE_METADATA_KEY

participant.metadata
# {
#     "tier": "gold",                                     # yours, untouched
#     "conference": {
#         "asserted":   {"sip.phoneNumber": "+15551234"},   # the SFU vouched
#         "unasserted": {"nickname": "bob"},                # the client said so
#     },
# }
participant.metadata[CONFERENCE_METADATA_KEY]["asserted"]
```

A re-join refreshes that one key and touches nothing else. Each bag keeps at most
32 attributes, keys up to 128 characters and values up to 1024 characters
serialized, first seen kept — so a participant that floods its own attributes can
evict neither what the SFU asserted about it nor what it was already carrying.

See `examples/conference_identity_provenance.py` for all of this end to end.

### What an arrival does not do

An arrival is not a message, so there is nothing to hold, refuse, or reply to. It
does **not** fire `ON_IDENTITY_AMBIGUOUS` / `ON_IDENTITY_UNKNOWN`, and it runs no
challenge or rejection flow — those act on an inbound message, and the
participant's first utterance goes through the full pipeline like any other.
An `AMBIGUOUS` or `PENDING` result is recorded as a pending identification with
its candidates; `UNKNOWN`, `REJECTED` and `CHALLENGE_SENT` leave the participant
unknown. `identity_timeout` applies as everywhere else: on timeout the result is
`UNKNOWN`, the framework event is emitted, and the participant joins regardless.
A resolver that raises never keeps someone out of a meeting.

### Turning it off

`identity_channel_types` gates the arrival exactly as it gates the inbound
pipeline. A deployment that restricts resolution — to hold a contractual or
data-processing limit on what leaves for an external resolver — is not bypassed
by the conference path: a dial-in's caller number is one of the addresses that
restriction exists to keep in.

```python
kit = RoomKit(
    identity_resolver=my_resolver,
    identity_channel_types={ChannelType.SMS},   # conferences excluded
)
```

The arrival is otherwise unchanged: the participant joins, the roster records
it, the provider's attributes stay on `Participant.metadata` — only the lookup
does not happen, and the participant stays `UNKNOWN`. Include
`ChannelType.CONFERENCE` in the set (or leave `identity_channel_types` at
`None`) to resolve dial-ins.

One accessor answers the question wherever it is asked:

```python
kit.identity_enabled_for(ChannelType.CONFERENCE)   # False, above
```

See RFC §12.10.2 for the normative rules.

## Hook Filtering

Identity hooks support the same filtering as regular hooks:

```python
@kit.identity_hook(
    HookTrigger.ON_IDENTITY_UNKNOWN,
    channel_types={ChannelType.SMS},       # Only SMS
    channel_ids={"sms-support"},           # Only this channel
)
async def sms_only_handler(event, context, id_result):
    return IdentityHookResult.pending(display_name="SMS User")
```

## Testing with MockIdentityResolver

```python
from __future__ import annotations

from roomkit.identity.base import Identity
from roomkit.identity.mock import MockIdentityResolver

alice = Identity(id="alice", display_name="Alice")
bob = Identity(id="bob", display_name="Bob")

resolver = MockIdentityResolver(
    mapping={
        "alice-phone": alice,      # Known sender → IDENTIFIED
    },
    ambiguous={
        "shared-phone": [alice, bob],  # Multiple matches → AMBIGUOUS
    },
    unknown_status=IdentificationStatus.UNKNOWN,  # Default for unrecognized senders
)

kit = RoomKit(identity_resolver=resolver)
```
