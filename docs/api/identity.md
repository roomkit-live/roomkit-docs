# Participants & Identity

::: roomkit.Participant

::: roomkit.Identity

::: roomkit.IdentityResult

::: roomkit.IdentityHookResult

::: roomkit.IdentityResolver

::: roomkit.MockIdentityResolver

## Identity Hooks

Identity hooks (`ON_IDENTITY_AMBIGUOUS`, `ON_IDENTITY_UNKNOWN`) allow custom handling when the identity resolver returns ambiguous or unknown results.

### Basic Usage

```python
from roomkit import RoomKit, HookTrigger, IdentityHookResult

kit = RoomKit(identity_resolver=my_resolver)

@kit.identity_hook(HookTrigger.ON_IDENTITY_AMBIGUOUS)
async def resolve_ambiguous(event, ctx, id_result):
    # Custom logic to resolve ambiguous identity
    if len(id_result.candidates) == 1:
        return IdentityHookResult.resolved(id_result.candidates[0])
    return IdentityHookResult.pending(candidates=id_result.candidates)
```

### Filtering Identity Hooks

Identity hooks support filtering by `channel_types`, `channel_ids`, and `directions`, just like regular hooks. This allows you to target identity resolution logic to specific channels.

```python
from roomkit import ChannelType, ChannelDirection, HookTrigger

# Only run for SMS inbound messages
@kit.identity_hook(
    HookTrigger.ON_IDENTITY_AMBIGUOUS,
    channel_types={ChannelType.SMS},
    directions={ChannelDirection.INBOUND},
)
async def resolve_sms_identity(event, ctx, id_result):
    # Only called for SMS inbound events
    return IdentityHookResult.pending(
        display_name=ctx.room.metadata.get("phone_number"),
        candidates=id_result.candidates,
    )

# Only run for specific channel IDs
@kit.identity_hook(
    HookTrigger.ON_IDENTITY_UNKNOWN,
    channel_ids={"sms-support", "sms-sales"},
)
async def handle_unknown_support(event, ctx, id_result):
    # Only called for support/sales SMS channels
    return IdentityHookResult.reject("Unknown sender not allowed")
```

### Filter Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `channel_types` | `set[ChannelType] \| None` | Only run for events from these channel types (None = all) |
| `channel_ids` | `set[str] \| None` | Only run for events from these channel IDs (None = all) |
| `directions` | `set[ChannelDirection] \| None` | Only run for events with these directions (None = all) |

All specified filters must match for the hook to be invoked. If a filter is `None`, it matches all values.

### Use Cases

**SMS-only identity resolution:**
```python
@kit.identity_hook(
    HookTrigger.ON_IDENTITY_AMBIGUOUS,
    channel_types={ChannelType.SMS, ChannelType.EMAIL},
    directions={ChannelDirection.INBOUND},
)
async def resolve_transport_identity(event, ctx, id_result):
    # Identity resolution for transport channels only
    # WebSocket/AI channels are already authenticated
    ...
```

**Channel-specific rejection:**
```python
@kit.identity_hook(
    HookTrigger.ON_IDENTITY_UNKNOWN,
    channel_ids={"sms-premium"},
)
async def reject_unknown_premium(event, ctx, id_result):
    # Premium channel requires known identity
    return IdentityHookResult.reject("Unknown senders not allowed on premium channel")
```

