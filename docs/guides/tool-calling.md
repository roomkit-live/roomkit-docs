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
    provider=AnthropicAIProvider(model="claude-sonnet-4-20250514", api_key="..."),
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
