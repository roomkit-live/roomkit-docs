# Human-in-the-Loop Tools

RoomKit's human-in-the-loop primitive lets AI tool calls **pause execution**, request input from a human user, and **resume with the answer**. This enables tools like `AskUserQuestion`, confirmation dialogs, and any workflow where the AI needs to collect information from a person before continuing.

---

## Why Human-in-the-Loop?

AI agents often need to ask clarifying questions, request approval, or collect data from users mid-conversation. Without a dedicated primitive, developers resort to workarounds:

- **Wrapping tool handlers** — only intercepts tools in the admin chain, not the AI's own tools
- **BEFORE_TOOL_USE hooks** — can allow/deny but can't provide a result or pause the loop
- **External state management** — manual `asyncio.Event` tracking scattered across REST endpoints

RoomKit's `HumanInputHandler` solves this as a first-class feature with two layers:

- **`HumanInputHandler`** — core async primitive for create / wait / resolve / reject
- **`HumanInputToolHandler`** — ToolHandler wrapper that composes into AIChannel's tool chain

---

## Quick Start

### Native AIChannel (built-in tools)

For AI agents that define their own human-input tools:

```python
from roomkit import RoomKit, HumanInputToolHandler, HookTrigger, HookExecution
from roomkit.channels.ai import AIChannel
from roomkit.providers.ai.anthropic import AnthropicAIProvider

kit = RoomKit()

# Define which tools need human input
human = HumanInputToolHandler(
    tool_names={"AskUserQuestion", "ConfirmAction"},
    timeout=300,  # 5 minutes
)

ai = AIChannel(
    "agent",
    provider=AnthropicAIProvider(model="claude-opus-5", api_key="..."),
    system_prompt="You are a helpful assistant. Use AskUserQuestion to ask the user.",
    tool_handler=my_other_tools,        # Your regular tools
    human_input_handler=human,           # Human-input tools
)

kit.register_channel(ai)

# React when the AI asks for human input
@kit.hook(HookTrigger.ON_USER_INPUT_REQUIRED, execution=HookExecution.SYNC)
async def notify_user(event, ctx):
    # Broadcast to frontend via WebSocket, REST, etc.
    await ws_manager.broadcast(event.room_id, {
        "type": "question_pending",
        "pending_id": event.pending_id,
        "tool_name": event.tool_name,
        "arguments": event.arguments,
    })

# When the user answers (from your REST endpoint, WebSocket handler, etc.)
human.handler.resolve(pending_id, '{"answer": "blue"}')
```

### External Provider (Claude Code)

For providers that execute tools internally (like Claude Code sandboxes), use `HumanInputHandler` directly inside your `ExternalToolHandler`:

```python
from roomkit.tools.human_input import HumanInputHandler

handler = HumanInputHandler()

# Inside your ExternalToolHandler.process_tool_call():
async def process_tool_call(self, tool_name, tool_input, *, room_id=None, **kw):
    if tool_name == "AskUserQuestion":
        pending = await handler.create(
            tool_name, tool_input,
            room_id=room_id or "",
            tool_call_id=kw.get("tool_call_id", ""),
            channel_id=self._channel_id,
        )
        # ON_USER_INPUT_REQUIRED hook fires via _on_input_required callback
        result = await handler.wait(pending.pending_id, timeout=300)
        return ToolDecision(approved=False, reason=result)
    return ToolDecision(approved=True)

# When user answers:
handler.resolve(pending_id, answer_json)
```

---

## How It Works

### Data Flow

```
AI calls AskUserQuestion(questions=[...])
  → HumanInputToolHandler intercepts (tool name matches)
  → HumanInputHandler.create() registers pending request — answerable from here
  → HumanInputHandler.wait() blocks (asyncio.Event)
    ⇄ alongside: ON_USER_INPUT_REQUIRED hooks run (sync, can deny)
        → App broadcasts "question_pending" to frontend via WebSocket
  → User sees question in UI, selects answer
  → App calls handler.resolve(pending_id, answer_json)
  → wait() unblocks, returns answer as tool result
  → AI continues with user's response
```

The notification runs **alongside** the wait, not in front of it. A user who answers while a slow broadcast is still fanning out — or while a hook is burning its 30-second budget — is answering a request that is already listening, and the tool resumes on the answer. A denial from an `ON_USER_INPUT_REQUIRED` hook still rejects the request, wherever `wait()` has got to.

### Hook Execution Order

Three hooks interact during a human-input tool call:

| Order | Hook | Type | Purpose |
|-------|------|------|---------|
| 1 | `BEFORE_TOOL_USE` | Sync | Gate: should this tool run at all? |
| 2 | `ON_USER_INPUT_REQUIRED` | Sync | Notify: broadcast question to user; can deny |
| 3 | `ON_TOOL_CALL` | Sync | Observe: tool completed with result |

`BEFORE_TOOL_USE` fires before the handler — it can deny the tool (e.g., rate limiting). `ON_USER_INPUT_REQUIRED` is scheduled by `create()` and keeps sync semantics — hooks run in priority order and a `HookResult.block()` rejects the request — but it does not hold up the wait. `ON_TOOL_CALL` fires after `wait()` returns — standard observability.

### Who Owns the Cleanup

`wait()` owns it. It retires the request when the answer, the rejection, or the timeout arrives — callers should not track requests on the side and drop them, because a request removed under `wait()`'s feet is an answer turned into an error.

For a runtime that owns its own tool loop and carries the answer back another way — nobody calls `wait()` — say so at the call site with `create_detached()`, and free the request with `release()`:

```python
pending = await handler.create_detached("AskUserQuestion", args, room_id=room_id)
# ... answer travels back through the runtime's own path ...
handler.release(pending.pending_id)
```

`release()` rejects a request that is still unanswered, so a stray waiter unblocks rather than hanging.

### Late and Repeated Reads

An outcome that has been consumed stays readable. `wait()` retires the request into a bounded retention — the last 128 outcomes by default, `HumanInputHandler(retention=…)` — and replays it: a second `wait()` returns the same answer, re-raises the same rejection, or re-raises the timeout. Only a genuinely unknown id — never seen, or evicted from the retention — raises `ValueError`.

### Parallel Tool Execution

When the AI calls multiple tools in one round, `asyncio.gather` runs them concurrently. If one tool is a human-input tool, it blocks while the others complete normally. The tool loop waits for **all** results before the next AI generation round.

---

## API Reference

### HumanInputHandler

The core primitive that manages pending requests:

| Method | Signature | Description |
|--------|-----------|-------------|
| `create` | `async (tool_name, arguments, *, room_id, tool_call_id, channel_id) → PendingInput` | Register a pending request and schedule the `_on_input_required` callback. `wait()` owns the cleanup |
| `create_detached` | `async (…same…) → PendingInput` | Same, for a request no one will `wait()` on. The caller owns the cleanup and MUST `release()` it |
| `wait` | `async (pending_id, *, timeout=300) → str` | Block until resolved/rejected/timeout; replays an outcome already consumed |
| `resolve` | `(pending_id, result) → bool` | Unblock with answer. Returns `True` if found |
| `reject` | `(pending_id, reason="") → bool` | Unblock with error. Returns `True` if found |
| `release` | `(pending_id) → bool` | Drop a request the caller owns; rejects it if still unanswered. Returns `True` if one was dropped |
| `pending` | `property → dict[str, PendingInput]` | Snapshot of active pending requests |

`HumanInputHandler(retention=128)` sets how many consumed outcomes stay replayable; `retention=0` switches the retention off.

### HumanInputToolHandler

ToolHandler wrapper for the native AIChannel path:

```python
human = HumanInputToolHandler(
    tool_names={"AskUserQuestion"},  # Tools that need human input
    timeout=300,                      # Seconds before timeout error
    handler=None,                     # Optional: share a HumanInputHandler
)
```

- Pass to `AIChannel(human_input_handler=human)` — auto-composes with the tool chain
- Access the inner handler via `human.handler` for `resolve()` / `reject()`
- Falls through for non-matching tools (works with `compose_tool_handlers`)

### PendingInput

Mutable dataclass representing a pending request:

| Field | Type | Description |
|-------|------|-------------|
| `pending_id` | `str` | Unique ID for resolving |
| `tool_name` | `str` | Name of the tool |
| `arguments` | `dict` | Tool arguments |
| `room_id` | `str` | Room ID |
| `tool_call_id` | `str` | Provider-assigned tool call ID |
| `channel_id` | `str` | Originating channel |
| `status` | `PendingInputStatus` | `PENDING` / `RESOLVED` / `REJECTED` / `TIMED_OUT` |
| `result` | `str \| None` | Answer (set on resolve) |
| `detached` | `bool` | No one will `wait()` on it — its creator frees it with `release()` |
| `created_at` | `datetime` | Timestamp |

### PendingInputEvent

Frozen dataclass passed to `ON_USER_INPUT_REQUIRED` hooks:

```python
@kit.hook(HookTrigger.ON_USER_INPUT_REQUIRED, execution=HookExecution.SYNC)
async def on_input(event, ctx):
    print(event.pending_id)    # ID for resolving
    print(event.tool_name)     # "AskUserQuestion"
    print(event.arguments)     # {"questions": [...]}
    print(event.room_id)       # Room context
```

---

## Timeout Handling

If the user doesn't respond within the timeout:

- `wait()` raises `TimeoutError`
- `HumanInputToolHandler` catches it and returns a JSON error to the AI:
  ```json
  {"error": "Human input timed out after 300s for tool 'AskUserQuestion'"}
  ```
- The AI sees the error and can retry, skip, or inform the user

---

## Full Example

See [`examples/ai_human_input.py`](https://github.com/anthropics/roomkit/blob/main/examples/ai_human_input.py) for a complete working example with:

- MockAIProvider simulating an `AskUserQuestion` tool call
- `HumanInputToolHandler` pausing the tool loop
- Async resolution simulating a user answering
- `ON_USER_INPUT_REQUIRED` hook for notifications
