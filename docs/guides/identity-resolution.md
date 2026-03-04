# Identity Resolution

Identity resolution maps inbound message senders to known participants. It runs automatically in the inbound pipeline — after message parsing and before broadcast hooks.

## Quick Start

```python
from __future__ import annotations

from roomkit import (
    Identity,
    IdentificationStatus,
    IdentityResolver,
    IdentityResult,
    RoomKit,
)
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
from roomkit import Identity

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

from roomkit import HookTrigger, IdentityHookResult, RoomKit

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

from roomkit import HookTrigger, IdentityHookResult
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
| `identity_channel_types` | `None` | Restrict to specific channel types. `None` = all |
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

from roomkit import Identity, MockIdentityResolver

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
