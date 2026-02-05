# MCP Integration

RoomKit is designed for seamless integration with the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/). Build AI assistants that can manage conversations, send messages, and handle multi-channel communication.

## What is MCP?

The Model Context Protocol is an open standard that enables AI assistants like Claude to interact with external tools and data sources. With RoomKit's MCP integration, AI assistants can:

- Create and manage conversation rooms
- Send messages across channels (SMS, Email, WebSocket, etc.)
- Query conversation history
- Manage participants and identities
- Handle real-time events

## Quick Start

### 1. Install RoomKit

```bash
pip install roomkit
```

### 2. Create an MCP Server

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
from roomkit import RoomKit

kit = RoomKit()
server = Server("roomkit-mcp")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="roomkit_create_room",
            description="Create a new conversation room",
            inputSchema={
                "type": "object",
                "properties": {
                    "metadata": {
                        "type": "object",
                        "description": "Optional metadata for the room"
                    }
                }
            }
        ),
        Tool(
            name="roomkit_send_message",
            description="Send a message to a room",
            inputSchema={
                "type": "object",
                "properties": {
                    "room_id": {"type": "string", "description": "Room ID"},
                    "body": {"type": "string", "description": "Message content"},
                    "channel_id": {"type": "string", "description": "Target channel"}
                },
                "required": ["room_id", "body"]
            }
        ),
        Tool(
            name="roomkit_list_rooms",
            description="List all conversation rooms",
            inputSchema={"type": "object", "properties": {}}
        ),
        Tool(
            name="roomkit_get_history",
            description="Get message history for a room",
            inputSchema={
                "type": "object",
                "properties": {
                    "room_id": {"type": "string", "description": "Room ID"},
                    "limit": {"type": "integer", "description": "Max messages"}
                },
                "required": ["room_id"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "roomkit_create_room":
        room = await kit.create_room(metadata=arguments.get("metadata"))
        return [TextContent(type="text", text=f"Created room: {room.id}")]

    elif name == "roomkit_send_message":
        # Implementation depends on your channel setup
        pass

    elif name == "roomkit_list_rooms":
        rooms = await kit.list_rooms()
        return [TextContent(type="text", text=f"Rooms: {[r.id for r in rooms]}")]

    elif name == "roomkit_get_history":
        events = await kit.get_room_events(
            arguments["room_id"],
            limit=arguments.get("limit", 50)
        )
        return [TextContent(type="text", text=str(events))]

async def main():
    async with stdio_server() as (read, write):
        await server.run(read, write)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 3. Configure Claude Desktop

Add to your Claude Desktop config (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "roomkit": {
      "command": "python",
      "args": ["/path/to/your/mcp_server.py"]
    }
  }
}
```

## Available Tools

A complete RoomKit MCP server typically exposes these tools:

| Tool | Description |
|------|-------------|
| `roomkit_create_room` | Create a new conversation room |
| `roomkit_list_rooms` | List all rooms with optional filtering |
| `roomkit_get_room` | Get details about a specific room |
| `roomkit_send_message` | Send a message to a room |
| `roomkit_get_history` | Retrieve conversation history |
| `roomkit_attach_channel` | Attach a channel to a room |
| `roomkit_add_participant` | Add a participant to a room |
| `roomkit_list_participants` | List room participants |

## Example Prompts

Once configured, you can interact with RoomKit through natural language:

**Creating rooms:**
> "Create a new support room for customer inquiries"

**Sending messages:**
> "Send 'Hello, how can I help you today?' to room rm_abc123"

**Querying history:**
> "Show me the last 10 messages in the support room"

**Managing channels:**
> "Attach the SMS channel to the customer's room"

## AI Context Files

RoomKit includes files specifically designed to help AI assistants understand the library:

### llms.txt

Provides a structured overview of RoomKit documentation for LLM context windows:

```python
from roomkit import get_llms_txt

# Include in your MCP server's context
context = get_llms_txt()
```

### AGENTS.md

Contains coding guidelines and patterns for AI assistants:

```python
from roomkit import get_agents_md

# Help AI write idiomatic RoomKit code
guidelines = get_agents_md()
```

## Best Practices

### 1. Use Meaningful Metadata

```python
room = await kit.create_room(metadata={
    "type": "support",
    "customer_id": "cust_123",
    "priority": "high"
})
```

### 2. Implement Proper Error Handling

```python
@server.call_tool()
async def call_tool(name: str, arguments: dict):
    try:
        # Tool implementation
        pass
    except RoomNotFoundError:
        return [TextContent(type="text", text="Error: Room not found")]
    except Exception as e:
        return [TextContent(type="text", text=f"Error: {str(e)}")]
```

### 3. Provide Rich Context

Include conversation context when the AI needs to make decisions:

```python
Tool(
    name="roomkit_analyze_conversation",
    description="Analyze conversation sentiment and suggest responses",
    inputSchema={
        "type": "object",
        "properties": {
            "room_id": {"type": "string"},
            "include_history": {"type": "boolean", "default": True}
        }
    }
)
```

## Resources

- [Model Context Protocol Documentation](https://modelcontextprotocol.io/)
- [RoomKit API Reference](/docs/api/)
- [AI Integration Guide](/docs/ai-integration/)
- [llms.txt Specification](https://llmstxt.org/)
