# Memory Provider

A pluggable memory backend that controls how conversation history is retrieved for AI context construction. By default, `AIChannel` uses a sliding window of the last N events. Custom providers can inject summaries, retrieve from vector stores, or combine multiple memory strategies.

## Quick start

```python
from roomkit import AIChannel
from roomkit.memory import SlidingWindowMemory
from roomkit.providers.anthropic.ai import AnthropicAIProvider

# Default — last 50 events (same as omitting memory entirely)
ai = AIChannel("ai", provider=provider, max_context_events=50)

# Explicit — equivalent to the default
ai = AIChannel("ai", provider=provider, memory=SlidingWindowMemory(max_events=50))
```

When no `memory` parameter is provided, `AIChannel` auto-creates a `SlidingWindowMemory` configured with `max_context_events`. Existing code continues to work unchanged.

## How it works

When an event triggers AI generation, `AIChannel` calls `memory.retrieve()` to get a `MemoryResult`. The result contains three optional fields:

- **`messages`** — Pre-built `AIMessage` objects (summaries, system context, synthetic messages). These are prepended to the context as-is.
- **`events`** — Raw `RoomEvent` objects. These are converted by `AIChannel` using its own content extraction logic, preserving vision support and role determination.
- **`notes`** — What the provider retrieved for the current turn alone (knowledge passages, say). They ride the turn's notes, after the current event, never the history: they change from one turn to the next, and in the history they would shift everything after them, which a provider caching a prefix bills again at every turn (RFC §20.2).

```
Event arrives → memory.retrieve() → MemoryResult
                                      ├── messages → prepended directly
                                      ├── events  → converted via _extract_content()
                                      │               ↓
                                      │      current event appended
                                      └── notes   → the turn's notes, after it
                                                     ↓
                                              AIContext → Provider
```

A provider whose `messages` name participants (a summary's lines, say) lists the labels it named them by in `speakers`, as `CompactingMemory` and `SummarizingMemory` do: the channel counts them toward labelling the turns that follow, so a summary that names several people is followed by labelled turns (RFC §6.4). A provider can return any of the fields. A provider that wraps another and rebuilds its result must carry the inner `notes`: build it with `dataclasses.replace(inner_result, ...)` rather than a new `MemoryResult(messages=..., events=...)`, which drops them. A provider that keeps the turn to a token budget counts them (`roomkit.memory.estimate_notes_tokens`): nothing trims them. `SlidingWindowMemory` returns only `events`. A summarization provider might return only `messages`. A hybrid could return both.

`context.recent_events` is the room's **tail**, in ascending order: its last
element is the most recent event. That is why providers slice `[-N:]` rather
than `[:N]` -- the head of a long room is the part you want to drop, not keep.

It is also already **this channel's view** of the room, not the room's whole
timeline. Events whose visibility excluded the AI channel were removed before
`retrieve()` was called, so your provider cannot accidentally put into a prompt
something the room decided that channel should not see -- including by
summarizing it. You do not filter, and you cannot opt out. The channel's own
past events are always kept, so an assistant bound with a narrow visibility
still recalls what it said.

## Built-in providers

### SlidingWindowMemory

Returns the most recent events from `context.recent_events`. This replicates the original hardcoded behavior.

```python
from roomkit.memory import SlidingWindowMemory

memory = SlidingWindowMemory(max_events=25)
ai = AIChannel("ai", provider=provider, memory=memory)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_events` | `50` | Maximum number of recent events to include |

### MockMemoryProvider

Records all calls and returns pre-configured results. Useful for testing.

```python
from roomkit.providers.ai.base import AIMessage
from roomkit.memory import MockMemoryProvider

mock = MockMemoryProvider(
    messages=[AIMessage(role="system", content="Summary of prior conversation")],
)
ai = AIChannel("ai", provider=provider, memory=mock)

# After processing an event:
assert len(mock.retrieve_calls) == 1
assert mock.retrieve_calls[0].room_id == "room-123"
```

## Custom providers

Implement `MemoryProvider` to plug in any context retrieval strategy:

```python
from roomkit import RoomContext, RoomEvent
from roomkit.providers.ai.base import AIMessage
from roomkit.memory import MemoryProvider, MemoryResult


class SummaryMemory(MemoryProvider):
    """Injects a conversation summary plus recent events."""

    def __init__(self, summarizer, recent_count: int = 5) -> None:
        self._summarizer = summarizer
        self._recent_count = recent_count

    @property
    def name(self) -> str:
        return "SummaryMemory"

    async def retrieve(
        self,
        room_id: str,
        current_event: RoomEvent,
        context: RoomContext,
        *,
        channel_id: str | None = None,
    ) -> MemoryResult:
        summary = await self._summarizer.summarize(room_id)
        return MemoryResult(
            messages=[AIMessage(role="system", content=summary)],
            events=context.recent_events[-self._recent_count :],
        )
```

The ABC also provides optional lifecycle methods:

```python
class StatefulMemory(MemoryProvider):
    async def retrieve(self, room_id, current_event, context, *, channel_id=None):
        ...  # Required — fetch context

    async def ingest(self, room_id, event, *, channel_id=None):
        ...  # Optional — update internal state as events arrive

    async def clear(self, room_id):
        ...  # Optional — clear stored memory for a room

    async def close(self):
        ...  # Optional — release resources (connections, models)
```

`ingest()`, `clear()`, and `close()` are concrete no-ops in the base class, so simple providers only need to implement `retrieve()`.

## MemoryResult

The `MemoryResult` dataclass controls what goes into the AI context:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `messages` | `list[AIMessage]` | `[]` | Pre-built messages prepended to context |
| `events` | `list[RoomEvent]` | `[]` | Raw events converted by AIChannel |
| `notes` | `list[str]` | `[]` | Retrieved for this turn alone; carried after the current event |

Messages appear first in the context, followed by converted events, followed by the current triggering event, which carries the turn's notes after its own text. This ordering ensures summaries and system context appear before the conversation, and what changes from turn to turn comes last.

## Auto-default behavior

| `memory` | `max_context_events` | Behavior |
|----------|---------------------|----------|
| not set | not set | `SlidingWindowMemory(max_events=50)` |
| not set | set | `SlidingWindowMemory(max_events=N)` |
| set | any | Uses the explicit provider; `max_context_events` is ignored |

## Lifecycle

- `retrieve()` is called on every AI generation (each `on_event`)
- `close()` is called when `AIChannel.close()` is called
- `ingest()` is not called automatically — stateful providers should use `AFTER_BROADCAST` hooks to call it

## Example

See [`examples/memory_provider.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/memory_provider.py) for a runnable demo showing default, summary, and custom memory providers.
