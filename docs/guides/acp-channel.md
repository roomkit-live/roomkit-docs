# ACP Agent Channel

`ACPChannel` connects a RoomKit room to an external coding agent through the
[Agent Client Protocol](https://agentclientprotocol.com/). RoomKit acts as the
ACP **client**; Claude Agent, Codex CLI, Gemini CLI, or another compatible coding
agent acts as the ACP **server**.

This is distinct from a future integration where an agent built with RoomKit
would itself be exposed as an ACP server.

## Installation

Install RoomKit with the ACP extra:

```bash
pip install "roomkit[acp]"
```

The extra currently targets the official Python SDK
`agent-client-protocol>=0.11.0,<0.12` and the stable ACP wire protocol v1. ACP
artifact/package versions and wire protocol versions are separate; RoomKit
validates the negotiated `protocolVersion`.

## Basic setup

Choose an agent from the
[official ACP agent registry](https://agentclientprotocol.com/get-started/agents)
and use the launch command documented by that agent or adapter:

```python
from pathlib import Path

from roomkit import ACPChannel, ChannelCategory, RoomKit

kit = RoomKit()

agent = ACPChannel(
    "coding-agent",
    # Replace this argument vector with the selected agent's ACP command.
    # RoomKit launches it directly, without a shell.
    command=["my-acp-agent", "--stdio"],
    cwd=Path("/srv/workspaces/my-project"),
)

kit.register_channel(agent)
await kit.create_room(room_id="coding-session")
await kit.attach_channel(
    "coding-session",
    "coding-agent",
    category=ChannelCategory.INTELLIGENCE,
)
```

`cwd` and every `additional_directories` entry must be absolute paths. A single
agent process is started lazily for the channel. Each Room receives a separate
ACP session, and prompts are serialized within that session. Different Rooms
can make progress concurrently through the same connection.

## Claude Code with the CLI channel

The repository includes a complete interactive example that connects a
`CLIChannel` to Claude Code through `ACPChannel`:

```bash
uv sync --extra acp --extra console

# Authenticate once with a Claude subscription.
npx -y @agentclientprotocol/claude-agent-acp@0.61.0 \
    --cli auth login --claudeai

uv run python examples/acp_claude_code.py \
    --workspace /path/to/project \
    --thinking-tokens 1024
```

The adapter command and version above match the current
[official ACP registry](https://agentclientprotocol.com/get-started/registry):
`@agentclientprotocol/claude-agent-acp@0.61.0`, with no additional ACP argument.
It requires Node.js 22 or newer. `ANTHROPIC_API_KEY` may be used instead of the
subscription login; the example forwards it to the restricted ACP subprocess
environment.

The example enables `CLIChannel(markdown=True)`, which uses Rich's live display
to rebuild the accumulated Markdown document on every text delta. Headings,
lists, emphasis, links, tables, and fenced code therefore remain formatted
while the response is still arriving. Tool start/completion events are
displayed inline, and the terminal asks before every requested operation. It
only selects the agent's one-time approval option.

RoomKit forwards each ACP text delta immediately; it does not wait for the
prompt to finish. The visible granularity still depends on the agent: when
Claude sends its final summary as one ACP chunk, RoomKit renders that chunk at
once instead of manufacturing a fake typewriter animation.

Recent Claude models may omit visible thought text in their adaptive-thinking
mode. The example therefore forwards `MAX_THINKING_TOKENS` through
`--thinking-tokens` (default: `1024`) so the adapter requests a visible fixed
thinking budget. Set it to `0` for lower latency with reasoning disabled:

```bash
uv run python examples/acp_claude_code.py --thinking-tokens 0
```

Its Room wiring follows the same pattern as the other CLI examples:

```python
cli = CLIChannel("you", show_thinking=True, markdown=True)
claude = ACPChannel(
    "claude-code",
    command=["npx", "-y", "@agentclientprotocol/claude-agent-acp@0.61.0"],
    cwd=workspace,
    external_tool_handler=TerminalPermissionHandler(),
)

kit.register_channel(cli)
kit.register_channel(claude)
await kit.attach_channel(room_id, "you")
await kit.attach_channel(
    room_id,
    "claude-code",
    category=ChannelCategory.INTELLIGENCE,
)
await cli.run(kit, room_id=room_id)
```

## Event mapping

| ACP update | RoomKit representation |
|---|---|
| Agent message chunk | Text delta in the normal response stream |
| Agent thought chunk | `ThinkingDeltaMarker` and ephemeral thinking events |
| Tool call start/terminal status | Persisted tool-call markers and ephemeral activity |
| Intermediate tool progress | Ephemeral `CUSTOM` event with `type="acp_tool_progress"` |
| Plan update | Ephemeral `CUSTOM` event with `type="acp_plan_update"` |
| Usage update | Ephemeral `CUSTOM` event with `type="acp_usage"` |

Generated text and tool activity continue through RoomKit's regular streaming
pipeline. Visibility, chain-depth limits, persistence, hooks, and re-broadcast
therefore behave like other intelligence-channel output.

## Permissions

ACP agents may ask the client to approve a tool. `ACPChannel` rejects these
requests by default. To approve selected operations, provide an
`ExternalToolHandler`:

```python
from roomkit import ACPChannel, ToolPolicy
from roomkit.tools import PolicyExternalToolHandler

permissions = PolicyExternalToolHandler(
    policy=ToolPolicy(
        allow=["Read *", "Search *"],
        deny=["*delete*", "*credential*"],
    )
)

agent = ACPChannel(
    "coding-agent",
    command=["my-acp-agent", "--stdio"],
    cwd="/srv/workspaces/my-project",
    external_tool_handler=permissions,
)
```

RoomKit injects `BEFORE_TOOL_USE` and `ON_TOOL_CALL` hooks into the handler when
the channel is registered. An approval chooses `allow_once` when the agent
offers it; RoomKit does not silently grant durable permissions.

!!! warning
    ACP v1 tool calls expose a human-readable title and a coarse tool kind, not
    one universal canonical tool name across agents. Treat title-based glob
    policies as adapter-specific. For sensitive environments, implement a
    custom `ExternalToolHandler` that validates the title and structured input,
    and keep the default-deny behavior.

Client-side filesystem and terminal callbacks are not implemented or advertised.
The external agent still operates according to its own sandbox and the
permission options it presents.

## Cancellation and lifecycle

```python
# Notify the agent that the active turn should stop.
cancelled = await agent.cancel("coding-session")

# Close one Room's ACP session but keep the process for other Rooms.
closed = await agent.close_session("coding-session")

# RoomKit.close() closes all sessions and the subprocess.
await kit.close()
```

Session identifiers are process-local in this first implementation. A restart
creates new sessions; persistent ACP load/resume is not yet enabled.

## Current scope

- Stable ACP protocol v1 over stdio
- Text and rich-text Room input
- Streaming text, thoughts, tool lifecycle, plans, and usage
- Permission requests through `ExternalToolHandler`
- Per-Room session isolation, cancellation, and deterministic cleanup

Not yet included:

- ACP server mode for RoomKit-built agents
- Client-side filesystem, terminal, or elicitation implementations
- Image/audio prompt blocks
- Persistent session resume
- Experimental ACP protocol versions or draft transports
