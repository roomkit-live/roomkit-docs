# Tools

Tool system for AI function calling. See the [Tool Calling & Policies guide](../guides/tool-calling.md) and [MCP Tool Provider guide](../guides/mcp-tool-provider.md) for usage examples.

::: roomkit.tools.ToolPolicy

::: roomkit.tools.RoleOverride

::: roomkit.tools.MCPToolProvider

::: roomkit.tools.compose_tool_handlers

## Per-Call Context

A tool handler receives only `(name, arguments)`. These accessors read the turn
it is running under from a contextvar — see
[What a handler knows about the call](../guides/tool-calling.md#what-a-handler-knows-about-the-call).

::: roomkit.tools.current_tool_room_id

::: roomkit.tools.current_tool_room

::: roomkit.tools.current_tool_actor_id

::: roomkit.tools.current_tool_allowed_names

::: roomkit.tools.current_tool_call

::: roomkit.tools.ToolCallContext

::: roomkit.tools.current_response_metadata

::: roomkit.tools.tool_turn_context

## Turn Response Metadata

The record every writer of a turn shares — see
[What a handler can tell the turn](../guides/tool-calling.md#what-a-handler-can-tell-the-turn).

::: roomkit.models.ResponseMetadata

## Human-in-the-Loop

::: roomkit.tools.HumanInputHandler

::: roomkit.tools.HumanInputToolHandler

::: roomkit.PendingInput

::: roomkit.PendingInputEvent

::: roomkit.PendingInputStatus
