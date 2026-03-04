# Advanced Memory Providers

RoomKit's memory system controls what conversation context the AI sees. Beyond the default `SlidingWindowMemory`, two advanced providers handle long conversations: `BudgetAwareMemory` (token-budget trimming) and `CompactingMemory` (summarize + trim).

## MemoryProvider ABC

```python
from __future__ import annotations

from abc import ABC, abstractmethod

from roomkit.memory import MemoryResult


class MemoryProvider(ABC):
    @abstractmethod
    async def retrieve(self, room_id, current_event, context, *, channel_id=None) -> MemoryResult:
        """Retrieve context for AI generation."""
        ...

    async def ingest(self, room_id, event, *, channel_id=None) -> None:
        """Ingest an event (optional, for stateful providers)."""

    async def clear(self, room_id) -> None:
        """Clear memory for a room (optional)."""

    async def close(self) -> None:
        """Release resources (optional)."""
```

### MemoryResult

```python
@dataclass
class MemoryResult:
    messages: list[AIMessage] = field(default_factory=list)  # Pre-built messages (summaries)
    events: list[RoomEvent] = field(default_factory=list)    # Raw events for conversion
```

- **messages** are prepended first in the AI context (e.g., conversation summaries)
- **events** are converted by AIChannel using its content extraction logic (preserves vision/images)
- Both fields are optional — a provider may populate one or both

## When to Use Each

| Provider | Conversation Length | Cost | Use Case |
|----------|-------------------|------|----------|
| `SlidingWindowMemory` | < 50 messages | None | Simple chatbots, short conversations |
| `BudgetAwareMemory` | 50-500 messages | None | Medium conversations, no AI cost for memory |
| `CompactingMemory` | 500+ messages | LLM calls | Long conversations, full context retention |

---

## SlidingWindowMemory (Default)

Returns the most recent N events. Stateless and zero-cost.

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.memory import SlidingWindowMemory

memory = SlidingWindowMemory(max_events=50)

ai = AIChannel(
    "ai-assistant",
    provider=provider,
    memory=memory,
)
```

!!! note
    When no memory provider is specified, AIChannel creates `SlidingWindowMemory(max_events=max_context_events)` by default.

```python
# These are equivalent:
ai = AIChannel("ai", provider=provider, max_context_events=50)
ai = AIChannel("ai", provider=provider, memory=SlidingWindowMemory(max_events=50))
```

---

## BudgetAwareMemory

Wraps any inner provider and trims events to fit a token budget. No LLM calls — pure algorithmic trimming.

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.memory import BudgetAwareMemory, SlidingWindowMemory

memory = BudgetAwareMemory(
    inner=SlidingWindowMemory(max_events=200),
    max_context_tokens=8000,
    safety_margin_ratio=0.15,   # Reserve 15% of budget
    min_events=3,               # Never drop below 3 events
)

ai = AIChannel("ai-assistant", provider=provider, memory=memory)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `inner` | required | Wrapped memory provider |
| `max_context_tokens` | required | Total token budget for context |
| `safety_margin_ratio` | `0.15` | Reserve this fraction of budget (15%) |
| `min_events` | `3` | Minimum events to preserve |

**How it works**:

1. Calls `inner.retrieve()` to get events
2. Effective budget = `max_context_tokens * (1 - safety_margin_ratio)`
3. If total tokens exceed budget, trims oldest events first
4. Never drops below `min_events`
5. Preserves pre-built messages from inner provider unchanged

**Token estimation**: ~1 token per 4 characters (rough heuristic via `estimate_tokens()`).

---

## CompactingMemory

Extends budget-aware trimming with AI-powered summarization of older events:

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.memory import CompactingMemory, SlidingWindowMemory
from roomkit.providers.ai.anthropic import AnthropicAIProvider

# Use a fast, cheap model for summarization
summarizer = AnthropicAIProvider(model="claude-haiku-4-5-20251001", api_key="...")

memory = CompactingMemory(
    inner=SlidingWindowMemory(max_events=200),
    provider=summarizer,
    max_context_tokens=8000,
    summary_ratio=0.10,              # 10% of budget for summaries
    safety_margin_ratio=0.15,        # 15% safety margin
    min_events=5,                    # Keep at least 5 recent events
    summary_cache_ttl_seconds=300.0, # Cache summaries for 5 minutes
)

ai = AIChannel("ai-assistant", provider=provider, memory=memory)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `inner` | required | Wrapped memory provider |
| `provider` | required | AI provider for summarization |
| `max_context_tokens` | required | Total token budget |
| `summary_ratio` | `0.10` | Fraction of budget allocated to summaries |
| `safety_margin_ratio` | `0.15` | Safety margin fraction |
| `min_events` | `5` | Minimum events before compacting |
| `summary_cache_ttl_seconds` | `300.0` | How long to cache summaries per room |

**How it works**:

1. Calls `inner.retrieve()` to get all events
2. If total tokens fit in budget → return as-is (no compacting)
3. If over budget:
    - Split events into **trimmed** (old) and **kept** (recent)
    - Summarize trimmed events via the AI provider
    - Inject summary as a pre-built message at the context start
    - Return: `[summary_message] + [kept_events]`
4. Summaries are cached per-room with TTL to avoid regenerating on every call

**Graceful degradation**: If summarization fails (provider error), a placeholder message is injected instead.

!!! tip "Choosing a Summarizer"
    Use a fast, inexpensive model (e.g., Claude Haiku) for summarization. The summary prompt focuses on decisions made, key findings, tool results, and errors.

---

## Custom Memory Provider

Implement `MemoryProvider` for custom logic (e.g., vector store retrieval):

```python
from __future__ import annotations

from roomkit.memory import MemoryProvider, MemoryResult
from roomkit.providers.ai.base import AIMessage


class VectorStoreMemory(MemoryProvider):
    """Retrieve relevant context from a vector store."""

    def __init__(self, vector_db, top_k: int = 5, recent_count: int = 10) -> None:
        self._vector_db = vector_db
        self._top_k = top_k
        self._recent_count = recent_count

    @property
    def name(self) -> str:
        return "VectorStoreMemory"

    async def retrieve(self, room_id, current_event, context, *, channel_id=None):
        # Get relevant past context via similarity search
        query = current_event.content.body if hasattr(current_event.content, "body") else ""
        relevant = await self._vector_db.search(query, top_k=self._top_k, room_id=room_id)

        # Build a context summary from relevant results
        summary = "\n".join(f"- {r.text}" for r in relevant)
        summary_msg = AIMessage(
            role="user",
            content=f"[Relevant context from conversation history]\n{summary}",
        )

        # Also include the most recent events for immediate context
        recent = context.recent_events[-self._recent_count:]

        return MemoryResult(messages=[summary_msg], events=recent)

    async def ingest(self, room_id, event, *, channel_id=None):
        # Index new events in the vector store
        text = event.content.body if hasattr(event.content, "body") else str(event.content)
        await self._vector_db.index(text, room_id=room_id, event_id=event.id)
```

## Multi-Channel Memory

The `channel_id` parameter enables different memory strategies per AI channel in the same room:

```python
from __future__ import annotations

from roomkit.memory import MemoryProvider, MemoryResult


class PerChannelMemory(MemoryProvider):
    @property
    def name(self) -> str:
        return "PerChannelMemory"

    async def retrieve(self, room_id, current_event, context, *, channel_id=None):
        if channel_id == "ai-summarizer":
            # This channel sees the full history
            return MemoryResult(events=context.recent_events[-200:])
        else:
            # Other channels see only recent events
            return MemoryResult(events=context.recent_events[-20:])
```

## Token Estimation Utilities

```python
from roomkit.memory.token_estimator import (
    estimate_context_tokens,
    estimate_message_tokens,
    estimate_tokens,
)

# Text estimation (~1 token per 4 chars)
tokens = estimate_tokens("Hello, how can I help?")  # → 6

# Message estimation (includes role overhead)
tokens = estimate_message_tokens(AIMessage(role="user", content="Hello"))  # → 5

# Full context estimation (system prompt + messages + tools)
tokens = estimate_context_tokens(ai_context)
```

!!! note
    These are rough estimates. For exact token counting, use the provider's tokenizer (e.g., `tiktoken` for OpenAI). The built-in estimator is designed for budget allocation, not exact measurement.

## Testing with MockMemoryProvider

```python
from __future__ import annotations

from roomkit.memory import MockMemoryProvider
from roomkit.providers.ai.base import AIMessage

mock = MockMemoryProvider(
    messages=[AIMessage(role="system", content="Previous conversation summary")],
    events=[event1, event2],
)

# After usage:
assert len(mock.retrieve_calls) == 1
assert mock.retrieve_calls[0].room_id == "room-1"
assert mock.retrieve_calls[0].channel_id == "ai-assistant"
assert not mock.closed
```

The mock tracks all `retrieve()`, `ingest()`, and `clear()` calls for assertion.
