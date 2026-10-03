# AI Steering Directives & Fallback Chains

Steering directives let you dynamically control an active AI tool loop from external code — hooks, other channels, or background tasks. Combined with fallback provider chains, they give you fine-grained control over AI behavior in production.

## Quick Start

```python
from __future__ import annotations

from roomkit.models.steering import Cancel, InjectMessage, UpdateSystemPrompt

# Get the AI channel
ai = kit.get_channel("ai-assistant")

# Cancel an active tool loop
ai.steer(Cancel(reason="user_left"))

# Inject context between tool rounds
ai.steer(InjectMessage(content="User just uploaded a document.", role="user"))

# Append to the system prompt
ai.steer(UpdateSystemPrompt(append="\nIMPORTANT: User is a VIP customer."))
```

## Directive Types

### Cancel

Stops the current tool loop. Sets a fast-path cancel event so the loop exits without waiting for the next round.

```python
Cancel(reason="timeout")       # reason is logged
Cancel(reason="user_disconnected")
```

### InjectMessage

Adds a message to the AI context between tool rounds. The AI sees it on the next generation call.

```python
InjectMessage(content="New data available: Q4 revenue is $2.3M", role="user")
InjectMessage(content="Remember to check permissions first.", role="system")
```

| Field | Default | Description |
|-------|---------|-------------|
| `content` | required | Message text to inject |
| `role` | `"user"` | Message role (`"user"` or `"system"`) |

### UpdateSystemPrompt

Appends text to the system prompt between tool rounds. Useful for dynamic context injection.

```python
UpdateSystemPrompt(append="\n\nUser language preference: French")
UpdateSystemPrompt(append="\n\nNew policy: all refunds require manager approval.")
```

## How It Works

Each AI tool loop creates an isolated `_ToolLoopContext` with its own steering queue and cancel event. Directives flow through two checkpoints:

```
┌─────────────────────────────────────────────────────┐
│                   Tool Loop                          │
│                                                      │
│  ┌──────────────────┐                                │
│  │ Checkpoint 1     │  Fast-path cancel check        │
│  │ (before generate)│  before expensive API call     │
│  └────────┬─────────┘                                │
│           ▼                                          │
│  ┌──────────────────┐                                │
│  │ AI Generation    │  Provider API call             │
│  └────────┬─────────┘                                │
│           ▼                                          │
│  ┌──────────────────┐                                │
│  │ Tool Execution   │  Run tools in parallel         │
│  └────────┬─────────┘                                │
│           ▼                                          │
│  ┌──────────────────┐                                │
│  │ Checkpoint 2     │  Drain full queue: apply       │
│  │ (after tools)    │  Cancel, Inject, UpdatePrompt  │
│  └────────┬─────────┘                                │
│           ▼                                          │
│      Next round...                                   │
└─────────────────────────────────────────────────────┘
```

- **Checkpoint 1**: Fast cancel check — avoids expensive API calls if already cancelled
- **Checkpoint 2**: Full queue drain — applies all pending directives to the context

The `steer()` method is safe to call from any async context. Cancel directives also set the fast-path event for immediate exit.

## Practical Patterns

### Cancel on User Disconnect

```python
from __future__ import annotations

from roomkit import HookTrigger
from roomkit.models.steering import Cancel

@kit.hook(HookTrigger.ON_SESSION_ENDED)
async def cancel_on_disconnect(event, ctx):
    ai = kit.get_channel("ai-assistant")
    # The channel serves every room it is bound to: stop this room's turns only.
    ai.steer(Cancel(reason="user_disconnected"), room_id=ctx.room.id)
```

### Inject Real-Time Context from Another Channel

```python
from __future__ import annotations

from roomkit import HookTrigger, ChannelType
from roomkit.models.steering import InjectMessage

@kit.hook(HookTrigger.BEFORE_BROADCAST, channel_types={ChannelType.SMS})
async def inject_sms_context(event, ctx):
    ai = kit.get_channel("ai-assistant")
    ai.steer(
        InjectMessage(
            content=f"SMS received from {event.source.participant_id}: {event.content.body}",
            role="user",
        ),
        room_id=event.room_id,
    )
```

### Dynamic VIP System Prompt

```python
from __future__ import annotations

from roomkit.models.steering import UpdateSystemPrompt

async def on_vip_detected(participant_id: str):
    ai = kit.get_channel("ai-assistant")
    ai.steer(UpdateSystemPrompt(
        append=f"\n\nUser {participant_id} is a VIP. Prioritize their requests."
    ))
```

### Target a Room or a Loop

One channel object serves every room it is bound to, so its running loops
can belong to several members' turns at once. A host acting for one room
addresses that room (RFC §21.3):

```python
# A member's Stop: every loop of their room, and no other room's
reached = ai.steer(Cancel(reason="stop pressed"), room_id="room-a")

# Another directive reaches the room's most recent loop
ai.steer(InjectMessage(content="The user attached a file."), room_id="room-a")

# Or one loop by its id
ai.steer(Cancel(reason="timeout"), loop_id="loop-abc123")
```

`steer()` returns how many loops it reached. With neither `room_id` nor
`loop_id`, the directive reaches the most recently started loop, whatever its
room; giving both raises `ValueError`. A loop is reachable once its turn has
started (its response stream is read): a directive that comes before reaches
none, and `steer()` returns 0. See `examples/ai_shared_agent_rooms.py`.

## Fallback Provider Chain

Configure a fallback AI provider for production resilience:

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.models.channel import RetryPolicy
from roomkit.providers.ai.anthropic import AnthropicAIProvider
from roomkit.providers.ai.openai import OpenAIAIProvider

primary = AnthropicAIProvider(model="claude-opus-5", api_key="...")
fallback = OpenAIAIProvider(model="gpt-4o", api_key="...")

ai = AIChannel(
    "ai-assistant",
    provider=primary,
    fallback_provider=fallback,
    retry_policy=RetryPolicy(max_retries=2, base_delay_seconds=1.0),
)
```

### Fallback Flow

```
Primary provider
  ├─ Success → return response
  ├─ Retryable error (5xx, timeout) → retry with backoff
  │     ├─ Retry succeeds → return response
  │     └─ All retries exhausted → try fallback
  │           ├─ Fallback succeeds → return response
  │           └─ Fallback fails → raise original error
  └─ Non-retryable error (4xx) → fail immediately (skip fallback)
```

Works for both streaming and non-streaming generation.

The fallback receives the rounds the primary already ran in the turn, in a
form its vendor takes. A reasoning block without a signature (DeepSeek, Qwen,
vLLM and every vendor but Anthropic) does not go to Anthropic, which refuses
it and takes the round without it. A round of the current turn none of whose
calls carries a thought signature goes to a Gemini 3 model as text (what the
primary said, each call, each result with its images), since Gemini 3 refuses
a call it did not sign in the current turn; an earlier turn's round keeps its
form. Both were measured on the vendors' APIs (2026-10-03).

## Retry Policy

```python
from roomkit.models.channel import RetryPolicy

policy = RetryPolicy(
    max_retries=3,
    base_delay_seconds=1.0,
    exponential_base=2.0,
    max_delay_seconds=30.0,
)
```

| Field | Default | Description |
|-------|---------|-------------|
| `max_retries` | `0` | Maximum retry attempts |
| `base_delay_seconds` | `1.0` | Initial delay between retries |
| `exponential_base` | `2.0` | Backoff multiplier |
| `max_delay_seconds` | `30.0` | Maximum delay cap |

Delay formula: `min(base_delay * exponential_base^attempt, max_delay)`

## Tool Loop Configuration

```python
ai = AIChannel(
    "ai",
    provider=provider,
    max_tool_rounds=50,             # Max tool loop iterations (default: 50)
    tool_loop_timeout_seconds=300,   # Hard timeout in seconds (default: 300)
    tool_loop_warn_after=25,         # Soft warning at this round count (default: 25)
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_tool_rounds` | `50` | Maximum iterations before forced stop |
| `tool_loop_timeout_seconds` | `300.0` | Hard timeout (seconds). `None` disables |
| `tool_loop_warn_after` | `25` | Log warning at this round count |

## Testing

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.models.steering import Cancel, InjectMessage
from roomkit.providers.ai.mock import MockAIProvider

ai = AIChannel("ai", provider=MockAIProvider(responses=["Hello"]))

# Verify steer enqueues correctly
# (requires an active loop — see test_ai_steering.py for full patterns)
```

!!! tip
    The `MockAIProvider` with preset responses is ideal for testing steering behavior without API calls. See `tests/test_channels/test_ai_steering.py` for comprehensive test patterns.
