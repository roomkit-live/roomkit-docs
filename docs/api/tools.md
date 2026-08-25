# Tools

Tool system for AI function calling. See the [Tool Calling & Policies guide](../guides/tool-calling.md) and [MCP Tool Provider guide](../guides/mcp-tool-provider.md) for usage examples.

::: roomkit.tools.policy.ToolPolicy

::: roomkit.tools.policy.RoleOverride

::: roomkit.tools.mcp.MCPToolProvider

::: roomkit.tools.compose.compose_tool_handlers

## Per-Call Context

A tool handler receives only `(name, arguments)`. These accessors read the turn
it is running under from a contextvar — see
[What a handler knows about the call](../guides/tool-calling.md#what-a-handler-knows-about-the-call).

::: roomkit.tools.context.current_tool_room_id

::: roomkit.tools.context.current_tool_actor_id

::: roomkit.tools.context.current_tool_allowed_names

::: roomkit.tools.context.current_tool_call

::: roomkit.tools.context.current_response_metadata

## Turn Response Metadata

The record every writer of a turn shares — see
[What a handler can tell the turn](../guides/tool-calling.md#what-a-handler-can-tell-the-turn).

::: roomkit.models.response_metadata.ResponseMetadata

## Human-in-the-Loop

::: roomkit.tools.human_input.HumanInputHandler

::: roomkit.tools.human_input.HumanInputToolHandler

::: roomkit.models.pending_input.PendingInput

::: roomkit.models.pending_input.PendingInputEvent

::: roomkit.models.pending_input.PendingInputStatus
