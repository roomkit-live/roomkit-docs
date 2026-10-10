# MCP Integration

RoomKit is an MCP **client**: its AI channels call tools that
[Model Context Protocol](https://modelcontextprotocol.io/) servers expose.
RoomKit does not ship an MCP server of its own.

## MCP tools in an AI channel

```bash
pip install "roomkit[mcp]"
```

`MCPToolProvider` connects to an MCP server, by URL or by starting it as a
command, discovers its tools, and hands them to an `AIChannel` as ordinary
RoomKit tools. `compose_tool_handlers` mixes them with your local tools:

```python
from roomkit import AIChannel
from roomkit.tools import MCPToolProvider, compose_tool_handlers

async with MCPToolProvider.from_url("http://localhost:8000/mcp") as mcp:
    handler = compose_tool_handlers(local_handler, mcp.as_tool_handler())
    ai = AIChannel("ai-assistant", provider=provider, tool_handler=handler)
    # register the channel, then attach it with mcp.get_tools_as_dicts() in its metadata
```

An MCP tool call passes the same gates as a local one: tool policies and
`ON_TOOL_CALL` hooks. The [MCP Tool Provider guide](guides/mcp-tool-provider.md) covers
transports, filtering, headers and the tool-name aliases.

## MCP servers for an ACP agent

An external agent that takes part in a room over ACP (Claude Code, for
example) brings its own MCP servers. `ACPChannel(mcp_servers=[...])` declares
them in the agent's session; they name servers on the agent's machine, not on
RoomKit's. See the [ACP Agent Channel guide](guides/acp-channel.md).

## Driving RoomKit from an MCP client

To let an MCP client such as Claude Desktop act on rooms, write a server with
the [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) whose
tools call RoomKit's public API: `kit.create_room()`, `kit.get_room()`,
`kit.attach_channel()`, `kit.get_timeline()`, `kit.process_inbound()`. Check
who the caller is before each call: the server acts with the rights you give
it.
