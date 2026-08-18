# AI Tool Calling

RoomKit supports AI tool calling (function calling) with per-room tool definitions, streaming tool loops, access control via tool policies, and MCP integration. This guide covers the full tool calling system.

## Quick Start

The recommended way to define tools is with the **Tool protocol** — each tool bundles its JSON schema definition with its handler in a single object. Pass tool objects directly to `AIChannel(tools=[...])` and definitions + handlers are extracted automatically:

```python
from __future__ import annotations

import json

from roomkit import RoomKit, Tool
from roomkit.channels import AIChannel
from roomkit.models.enums import ChannelCategory
from roomkit.providers.ai.anthropic import AnthropicAIProvider


class GetWeatherTool:
    """Implements the Tool protocol: definition + handler."""

    @property
    def definition(self) -> dict:
        return {
            "name": "get_weather",
            "description": "Get current weather for a city",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"},
                    "units": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["city"],
            },
        }

    async def handler(self, name: str, arguments: dict) -> str:
        city = arguments["city"]
        return json.dumps({"temp": 22, "condition": "sunny", "city": city})


kit = RoomKit()
ai = AIChannel(
    "ai-assistant",
    provider=AnthropicAIProvider(model="claude-opus-5", api_key="..."),
    system_prompt="You are a helpful assistant.",
    tools=[GetWeatherTool()],
)
kit.register_channel(ai)

await kit.attach_channel("room-1", "ai-assistant", category=ChannelCategory.INTELLIGENCE)
```

No separate `tool_handler` or binding metadata `"tools"` list needed — the channel extracts both from the tool objects. When multiple tools are passed, their handlers are composed automatically with first-match-wins dispatch.

## Defining Tools

### AITool Model

```python
from roomkit.providers.ai.base import AITool

tool = AITool(
    name="get_weather",
    description="Get current weather for a city",
    parameters={
        "type": "object",
        "properties": {
            "city": {"type": "string", "description": "City name"},
            "units": {"type": "string", "enum": ["celsius", "fahrenheit"]},
        },
        "required": ["city"],
    },
)
```

### As Dicts in Binding Metadata

Tools can also be defined as plain dicts in channel binding metadata — they are automatically converted to `AITool` instances:

```python
await kit.attach_channel("room-1", "ai-assistant", metadata={
    "tools": [
        {"name": "search", "description": "Search the knowledge base", "parameters": {...}},
        {"name": "create_ticket", "description": "Create a support ticket", "parameters": {...}},
    ],
})
```

### Hub Tools and Hoisted Arguments

A *hub tool* declares one tool per domain behind an `{action, params}` signature:

```python
BOARDS_TOOL = {
    "name": "boards",
    "description": "Board operations.",
    "parameters": {
        "type": "object",
        "additionalProperties": False,
        "properties": {"action": {"type": "string"}, "params": {"type": "object"}},
        "required": ["action"],
    },
}
```

Models trained mostly on flat schemas (one tool = its arguments) routinely
*hoist* the inner keys one level up — the smaller the model, the more often:

```json
{"action": "list_columns", "board_id": "1a0a495f"}
```

The schema is closed, so the argument gate would refuse `board_id` and the turn
would be spent on an error the model can only fix by re-issuing the call.
RoomKit folds the call back into shape instead, before validation, on both the
AI and realtime voice channels — the handler receives
`{"action": "list_columns", "params": {"board_id": "1a0a495f"}}` and the round
does real work. Each fold is logged at INFO with the tool and the model id, so
the frequency of the case stays measurable per model.

The repair is deliberately narrow. It applies only when the schema closed
itself, declares a `params` property of type `object`, and carries at least one
undeclared root key — and only when `params` is absent or empty:

| Call | Outcome |
|------|---------|
| `{"action": "x", "board_id": "1"}` | folded into `params` |
| `{"action": "x", "params": {}, "board_id": "1"}` | folded into `params` |
| `{"action": "x", "params": {"a": 1}, "board_id": "1"}` | **refused** — both forms at once is ambiguous; the error says to pass every argument inside `params` |
| `{"city": "Laval", "units": "metric"}` on a flat tool | **refused** — no container to fold into, so an unknown argument stays an error |
| any call against a schema without `additionalProperties: false` | untouched — undeclared root keys are already legal there |

Arguments rewritten by a `BEFORE_TOOL_USE` hook are validated but never folded:
a flat payload out of a hook is that hook's bug, and naming it beats reshaping
it silently.

Opening the schema (`additionalProperties: true`) would make the error go away
too — and make a genuine typo silent, handing the tool an argument nobody
reads. The schema stays closed.

## Tool Handlers (Advanced)

For most use cases, the `Tool` protocol (shown above) is the recommended approach. The `tool_handler` parameter is available for advanced scenarios: MCP integration, custom auditing/logging wrappers, or dynamic dispatch logic that doesn't fit the per-tool-object model.

A tool handler is an async function that receives the tool name and arguments, and returns a string result:

```python
from __future__ import annotations

import json


async def my_handler(name: str, arguments: dict) -> str:
    if name == "get_weather":
        city = arguments["city"]
        # Call your weather API
        return json.dumps({"temp": 22, "condition": "sunny"})
    if name == "search":
        query = arguments["query"]
        # Search your knowledge base
        return json.dumps({"results": ["result1", "result2"]})
    return json.dumps({"error": f"Unknown tool: {name}"})


ai = AIChannel("ai", provider=provider, tool_handler=my_handler)
```

When both `tools` and `tool_handler` are provided, the channel merges them — Tool object handlers are tried first, then the explicit `tool_handler`.

!!! tip
    Return `json.dumps({"error": f"Unknown tool: {name}"})` for unrecognized tools. This pattern enables tool handler composition (see below).

## What a Handler Knows About the Call

The handler protocol is `(name, arguments) -> str` — no room, no speaker, no
toolset. That omission is deliberate: an `AIChannel` object is registered once
per `channel_id` and shared by every room it serves, so anything a handler
closed over when it was built describes whoever attached it, not the turn now
running. Three accessors read the current turn from a contextvar instead:

| Accessor | Answers |
|----------|---------|
| `current_tool_room_id()` | Which room this turn belongs to |
| `current_tool_actor_id()` | Whose turn it is — the participant id of the event that woke the channel |
| `current_tool_allowed_names()` | Every tool name the turn resolved, so a call is validated against the live toolset rather than an attach-time snapshot |

Contextvars propagate down the async call chain, so they work at any depth
without a signature change. Each returns `None` outside a tool loop (realtime
voice pipelines, direct calls) — keep your own fallback for those paths.

```python
from roomkit.tools import current_tool_actor_id, current_tool_room_id


async def my_handler(name: str, arguments: dict) -> str:
    room_id = current_tool_room_id()
    actor_id = current_tool_actor_id()
    ...
```

### The Actor Names the Turn, It Does Not Authenticate It

`current_tool_actor_id()` returns a room `Participant.id`. The inbound pipeline
substitutes the resolved `Identity.id` for it only once identification
succeeds — a sender still pending, ambiguous or unknown keeps whatever the
channel supplied, or a synthetic `pending-…`, and reads back just as
non-`None`. Reaching a person's rows with the raw value trades one wrong
principal (whoever attached the handler) for another (whoever the channel
claimed). Resolve it against the roster first:

```python
import json

from roomkit.models.enums import IdentificationStatus
from roomkit.tools import current_tool_actor_id, current_tool_room_id


async def my_handler(name: str, arguments: dict) -> str:
    room_id = current_tool_room_id()
    actor_id = current_tool_actor_id()
    if room_id is None or actor_id is None:
        return json.dumps({"error": "No turn to act for"})

    participant = await kit.store.get_participant(room_id, actor_id)
    if participant is None or participant.identification is not IdentificationStatus.IDENTIFIED:
        return json.dumps({"error": "Sender not identified"})

    return await fetch_rows_for(participant.identity_id)
```

The author need not be human, either: in a multi-agent room the waking event may
be another agent's, and its participant id reads back the same way — compare
`participant.role` against `ParticipantRole.AGENT` when that matters.

!!! warning
    `None` is an answer, not a missing value. A system injection, a webhook or a
    scheduled run has no author; falling back to whoever spoke last is how a
    tool answers one person with another person's data. Refuse, or use a
    principal you configured on purpose.

See [Identity Resolution](identity-resolution.md) for how a sender becomes an
identified participant, and `examples/tool_call_context.py` for a runnable
two-speaker room.

## Per-Room Tool Binding

Tools, system prompts, and temperature can be configured per-room via binding metadata:

```python
# Room 1: Weather assistant
await kit.attach_channel("room-1", "ai-assistant", metadata={
    "system_prompt": "You are a weather assistant.",
    "temperature": 0.3,
    "tools": [weather_tool_dict],
})

# Room 2: Support assistant with different tools
await kit.attach_channel("room-2", "ai-assistant", metadata={
    "system_prompt": "You are a support agent.",
    "temperature": 0.7,
    "max_tokens": 2048,
    "thinking_budget": 5000,
    "tools": [search_tool_dict, ticket_tool_dict],
})
```

| Metadata Key | Type | Description |
|-------------|------|-------------|
| `tools` | `list[dict]` | Tool definitions (JSON Schema format) |
| `system_prompt` | `str` | Override the channel's default system prompt |
| `temperature` | `float` | Override the channel's default temperature |
| `max_tokens` | `int` | Override max output tokens |
| `thinking_budget` | `int` | Override thinking budget tokens |

## Tool Policy (Access Control)

Control which tools are available to which roles:

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.tools.policy import RoleOverride, ToolPolicy

policy = ToolPolicy(
    allow=["get_weather", "search_*"],  # Glob patterns
    deny=["delete_*"],                   # Always blocked
    role_overrides={
        "supervisor": RoleOverride(
            allow=["delete_*"],          # Supervisors can delete
            mode="replace",              # Fully override base policy
        ),
        "intern": RoleOverride(
            allow=["search_*"],          # Interns can only search
            mode="restrict",             # Intersect with base (default)
        ),
    },
)

ai = AIChannel("ai", provider=provider, tools=[weather_tool, search_tool], tool_policy=policy)
```

### Resolution Rules

1. Empty allow AND empty deny → permit all (backward compatible)
2. If tool matches any deny pattern → **blocked**
3. If allow is non-empty and tool matches NO allow pattern → **blocked**
4. Otherwise → **permitted**

### Override Modes

| Mode | Behavior |
|------|----------|
| `restrict` (default) | Deny lists union, allow lists intersect (dual-constraint) |
| `replace` | Override completely replaces the base policy |

Patterns use `fnmatch` glob syntax: `search_*`, `mcp_*`, `tool_?`.

!!! note
    Skill infrastructure tools (`activate_skill`, `read_skill_reference`, `run_skill_script`) are never filtered by policy — they must always remain visible.

## MCP Tool Provider

Integrate tools from an MCP (Model Context Protocol) server:

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.tools.mcp import MCPToolProvider

async with MCPToolProvider.from_url("http://localhost:8000/mcp") as mcp:
    tools = mcp.get_tools()              # list[AITool]
    handler = mcp.as_tool_handler()      # ToolHandler

    ai = AIChannel("ai", provider=provider, tool_handler=handler)

    # Bind tools to a room
    await kit.attach_channel("room-1", "ai", metadata={
        "tools": mcp.get_tools_as_dicts(),
    })
```

### MCPToolProvider Options

```python
MCPToolProvider(
    url="http://localhost:8000/mcp",
    transport="streamable_http",      # or "sse"
    tool_filter=lambda name: not name.startswith("internal_"),
    headers={"Authorization": "Bearer ..."},
)
```

## Composing Multiple Handlers

When you pass multiple `Tool` objects to `tools=[...]`, their handlers are composed automatically — no manual composition needed.

For advanced cases where you have raw `ToolHandler` callables (e.g., MCP handlers, custom dispatchers), use `compose_tool_handlers` to chain them with first-match-wins dispatch:

```python
from __future__ import annotations

from roomkit.tools.compose import compose_tool_handlers

local_handler = my_local_handler
mcp_handler = mcp.as_tool_handler()

combined = compose_tool_handlers(local_handler, mcp_handler)
# local_handler is tried first; if it returns "Unknown tool: ...", mcp_handler is tried
```

The composition checks for `{"error": "Unknown tool: ..."}` in the JSON response. Any other response (including other errors) is treated as a valid result and returned immediately.

## Streaming Tool Calls

When `streaming=True` (default), tool calls are processed through the streaming tool loop:

```python
ai = AIChannel(
    "ai",
    provider=provider,
    tools=[my_tool],       # or tool_handler=handler for advanced use
    streaming=True,        # Default — enables streaming tool loop
)
```

The streaming loop emits `StreamEvent` objects: `StreamTextDelta`, `StreamThinkingDelta`, `StreamToolCall`, and `StreamDone`. Tools are executed concurrently via `asyncio.gather()`.

## Tool Call Events

AIChannel automatically publishes ephemeral `TOOL_CALL_START` and `TOOL_CALL_END` events that you can subscribe to:

```python
from __future__ import annotations

from roomkit.realtime import EphemeralEvent, EphemeralEventType


async def on_tool_event(event: EphemeralEvent) -> None:
    if event.type == EphemeralEventType.TOOL_CALL_START:
        tools = event.data["tool_calls"]
        print(f"Calling: {[t['name'] for t in tools]}")
    elif event.type == EphemeralEventType.TOOL_CALL_END:
        print(f"Completed in {event.data.get('duration_ms')}ms")


sub_id = await kit.subscribe_room("room-1", on_tool_event)
```

## Cross-turn tool memory

The AI context rebuilt for each turn contains MESSAGE events only — tool-call
events are filtered out (providers track tool context *within* a turn, not
across turns). Left alone, the model would lose all trace of the tools it
invoked from one turn to the next: it couldn't tell which tool or source it
already used, and under Tool Search it couldn't re-call a tool it used a moment
ago because the catalogue is re-hidden every turn.

`AIChannel` closes both gaps automatically with a per-room, in-memory record of
tool usage — no configuration needed:

- **A "what you did" digest** — a compact summary of recent tool calls (name +
  arguments + a short result preview) is appended to the system prompt, so the
  model knows what it already did. Bounded by recent *calls*.
- **Sticky re-exposure** — the distinct tool names called recently are
  re-revealed each turn, so a tool used once stays callable even while Tool
  Search hides the rest of the catalogue. Bounded by recent distinct *tools* —
  the conversation's working set — since this is the part that costs full tool
  schemas.

The record is in-memory and scoped per room: a process restart clears it (the
model simply rediscovers tools on next use), which is fine for continuity
within a live conversation.

## Tool Loop Configuration

```python
ai = AIChannel(
    "ai",
    provider=provider,
    tools=[my_tool],
    max_tool_rounds=200,            # Max iterations (default: 200)
    tool_loop_timeout_seconds=300,   # Hard timeout in seconds (default: 300)
    tool_loop_warn_after=50,         # Soft warning threshold (default: 50)
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_tool_rounds` | `200` | Maximum tool loop iterations before forced stop |
| `tool_loop_timeout_seconds` | `300.0` | Hard timeout for entire loop. `None` disables |
| `tool_loop_warn_after` | `50` | Log warning at this round count |

!!! warning
    Tool results are truncated at ~30K tokens to prevent context overflow. Very large tool results will be automatically trimmed.

## Concurrent Tool Execution

When the AI requests multiple tool calls in a single round, they are executed concurrently via `asyncio.gather()`:

```python
# If the AI calls get_weather("Paris") and get_weather("London") simultaneously:
# Both execute in parallel, results returned together
```

Each tool call is independently subject to:
1. **Policy check** — blocked tools return an error message
2. **Skill gating** — tools from unactivated skills are blocked
3. **Telemetry** — each call gets its own `SpanKind.LLM_TOOL_CALL` span

## Testing

Use `MockAIProvider` for deterministic tool calling tests:

```python
from __future__ import annotations

from roomkit.channels import AIChannel
from roomkit.providers.ai.mock import MockAIProvider

# MockAIProvider can return tool calls and then final responses
provider = MockAIProvider(responses=["The weather in Paris is 22C and sunny."])

ai = AIChannel("ai", provider=provider, tools=[GetWeatherTool()])
```
