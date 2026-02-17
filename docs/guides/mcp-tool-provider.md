# MCP Tool Provider

`MCPToolProvider` bridges [MCP](https://modelcontextprotocol.io/) servers into RoomKit's `AITool` / `ToolHandler` system. It discovers tools from a remote MCP server and exposes them as standard RoomKit tools that plug directly into `AIChannel`. The companion `compose_tool_handlers` utility chains multiple tool handlers so you can mix MCP tools with local tools in a single channel.

## Quick start

```python
from roomkit import AIChannel, compose_tool_handlers
from roomkit.tools import MCPToolProvider

# Connect to an MCP server and discover tools
async with MCPToolProvider.from_url("http://localhost:8000/mcp") as mcp:
    # Combine MCP tools with a local handler
    handler = compose_tool_handlers(local_handler, mcp.as_tool_handler())

    ai = AIChannel(
        "ai-assistant",
        provider=provider,
        tool_handler=handler,
    )
    # ... register channel, create room, attach with mcp.get_tools_as_dicts() in metadata
```

## How it works

```
MCP Server (remote)
    │
    ▼
MCPToolProvider
    ├── get_tools()           → list[AITool]     (for binding metadata)
    ├── as_tool_handler()     → ToolHandler       (for AIChannel)
    └── call_tool(name, args) → str               (direct invocation)
    │
    ▼
compose_tool_handlers(local_handler, mcp_handler)
    │
    ▼
AIChannel(tool_handler=composed)
```

`MCPToolProvider` connects to an MCP server via streamable HTTP or SSE, discovers available tools, and maps them to RoomKit's `AITool` model. The `as_tool_handler()` method returns a `ToolHandler` callable that routes tool calls to the MCP server.

When a tool is not recognized by a handler (it returns `{"error": "Unknown tool: ..."}`), `compose_tool_handlers` tries the next handler in the chain. The last handler's result is always returned as-is.

## MCPToolProvider

### Connection lifecycle

`MCPToolProvider` uses an async context manager. Tools are discovered on entry:

```python
from roomkit.tools import MCPToolProvider

provider = MCPToolProvider.from_url(
    "http://localhost:8000/mcp",
    transport="streamable_http",  # or "sse"
    headers={"Authorization": "Bearer token"},
    tool_filter=lambda name: name.startswith("search_"),
)

async with provider as mcp:
    print(mcp.tool_names)       # ["search_web", "search_docs"]
    tools = mcp.get_tools()     # list[AITool]
    handler = mcp.as_tool_handler()
    # ... use handler with AIChannel
# Connection is closed automatically
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `url` | required | MCP server URL |
| `transport` | `"streamable_http"` | Transport protocol: `"streamable_http"` or `"sse"` |
| `tool_filter` | `None` | Predicate to include only matching tool names |
| `headers` | `None` | HTTP headers sent with every request |

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `get_tools()` | `list[AITool]` | Discovered tools as RoomKit AITool objects |
| `get_tools_as_dicts()` | `list[dict]` | Tools as plain dicts for binding metadata |
| `tool_names` | `list[str]` | Names of all discovered tools |
| `call_tool(name, args)` | `str` | Call a tool directly and get the result |
| `as_tool_handler()` | `ToolHandler` | Get a handler for `AIChannel(tool_handler=...)` |

### Result serialization

MCP tool results are serialized to strings for RoomKit's `ToolHandler` protocol:

| MCP result | Serialized as |
|------------|---------------|
| Single `TextContent` | Plain text string |
| Multiple content parts | JSON array of strings |
| `isError=True` | `{"error": "..."}` |

### Installation

`MCPToolProvider` requires the `mcp` package:

```bash
pip install roomkit[mcp]
```

The import is lazy — `mcp` is only required when you actually connect.

## compose_tool_handlers

Chains two or more `ToolHandler` callables into a single handler with first-match-wins semantics:

```python
from roomkit import compose_tool_handlers

async def weather_handler(name: str, arguments: dict) -> str:
    if name == "get_weather":
        return '{"temp": 20, "city": "Montreal"}'
    return '{"error": "Unknown tool: ' + name + '"}'

async def math_handler(name: str, arguments: dict) -> str:
    if name == "add":
        return str(arguments["a"] + arguments["b"])
    return '{"error": "Unknown tool: ' + name + '"}'

combined = compose_tool_handlers(weather_handler, math_handler)

await combined("get_weather", {"city": "Montreal"})  # → weather_handler
await combined("add", {"a": 1, "b": 2})              # → math_handler
await combined("unknown", {})                         # → math_handler (last handler)
```

### How dispatch works

1. Each handler is called in order
2. If the result is `{"error": "Unknown tool: ..."}`, the next handler is tried
3. Any other result (including other errors) is returned immediately
4. The last handler's result is always returned, even if it's an unknown-tool error

This convention lets each handler signal "not my tool" with the standard error format, enabling clean composition.

## Full example: MCP + local tools

```python
import asyncio
import json
from roomkit import (
    AIChannel, ChannelCategory, InboundMessage, RoomKit,
    TextContent, WebSocketChannel, compose_tool_handlers,
)
from roomkit.tools import MCPToolProvider


async def local_handler(name: str, arguments: dict) -> str:
    if name == "get_time":
        return json.dumps({"time": "2026-02-16T12:00:00"})
    return json.dumps({"error": f"Unknown tool: {name}"})


async def main():
    async with MCPToolProvider.from_url("http://localhost:8000/mcp") as mcp:
        handler = compose_tool_handlers(local_handler, mcp.as_tool_handler())

        kit = RoomKit()
        ai = AIChannel("ai", provider=provider, tool_handler=handler)
        kit.register_channel(ai)

        await kit.create_room(room_id="demo")
        await kit.attach_channel("demo", "ai",
            category=ChannelCategory.INTELLIGENCE,
            metadata={
                "tools": [
                    {"name": "get_time", "description": "Get current time", "parameters": {}},
                    *mcp.get_tools_as_dicts(),
                ],
            },
        )
        # ... process messages
        await kit.close()
```

## Testing

Use mock sessions to test without a real MCP server:

```python
from roomkit.tools import MCPToolProvider

provider = MCPToolProvider("http://fake:8000/mcp")
provider._connected = True
provider._session = your_mock_session
# Populate provider._tools and provider._tool_set manually
# Then test get_tools(), call_tool(), as_tool_handler() as usual
```

See [`tests/test_mcp_tool_provider.py`](https://github.com/roomkit-live/roomkit/blob/main/tests/test_mcp_tool_provider.py) for complete mock patterns.

## Example

See [`examples/mcp_tool_provider.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/mcp_tool_provider.py) for a runnable demo showing `compose_tool_handlers` combining local and MCP-style tools with `AIChannel`.
