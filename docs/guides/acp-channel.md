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

## Environment passthrough

The ACP SDK spawns the agent with a deliberately trimmed environment —
`HOME`, `LOGNAME`, `PATH`, `SHELL`, `TERM`, `USER` only (the MCP practice) —
plus whatever you pass in `env=`. That default silently breaks tooling a
coding agent relies on: without `SSH_AUTH_SOCK`, for instance, every
git-over-SSH operation the agent runs falls back to the on-disk key files
and prompts for their passphrases **on the controlling terminal**, stealing
keystrokes from whatever else is reading it.

`inherit_env=` names parent-process variables to forward. Values are read at
each process spawn (a reconnect picks up a rotated agent socket), unset
names are skipped, and explicit `env=` entries win over inherited ones.
Nothing is forwarded by default:

```python
agent = ACPChannel(
    "coding-agent",
    command=["my-acp-agent", "--stdio"],
    cwd=Path("/srv/workspaces/my-project"),
    env={"MAX_THINKING_TOKENS": "1024"},      # explicit values
    inherit_env=["SSH_AUTH_SOCK", "LANG"],    # forwarded from the parent
)
```

Keep the list minimal — the trimmed default exists so the agent does not
inherit secrets it has no business seeing.

## Reaching an agent you cannot spawn

`command=` spawns the agent here and speaks over its stdio. When the agent runs
somewhere this process cannot start it — on a user's own machine behind a
relay, in another container — pass a `transport=` instead. ACP does not care
what carries it: the SDK builds its connection from a reader/writer pair, so a
transport is whatever produces one.

```python
import acp
from roomkit import ACPChannel, ACPTransport


class MyRelayTransport(ACPTransport):
    @property
    def name(self) -> str:
        return "my-relay"                      # shows up in channel.info

    async def open(self, client, *, queue):
        reader, writer = await connect_to_my_relay()
        return acp.connect_to_agent(client, writer, reader, queue=queue)

    async def close(self) -> None:
        ...                                    # must not raise

    def is_alive(self) -> bool:
        return not self._socket.closed         # optional; defaults to True


agent = ACPChannel(
    "coding-agent",
    transport=MyRelayTransport(),
    cwd="/srv/workspaces/my-project",          # a path on the AGENT's machine
)
```

`command` and `transport` are mutually exclusive, and one is required. `env` and
`inherit_env` configure the subprocess spawn, so they are refused next to a
transport rather than quietly ignored — a custom transport carries its own
environment to wherever the agent lives.

Only the pipe is yours. `initialize`, protocol-version negotiation,
`authenticate`, per-Room sessions, prompts, cancellation, permissions, config
options and the whole event mapping stay on the channel, so a transport inherits
them without reimplementing anything. Note the split on the session fields:
`cwd`, `additional_directories` and `mcp_servers` are declared to
`session/new`, which means they name paths and servers on the **agent's**
machine — the absolute-path check still applies, but the path does not have to
exist here.

`is_alive()` is what lets the channel notice a dead connection: when it turns
false, the next prompt reconnects and drops every session behind the old one (a
reconnect never resumes them). Answer only when you know — the default `True`
costs at worst a failed request the channel already reports, while a wrong
`False` throws away live sessions.

## Session config: model, mode, effort

ACP agents advertise their tunables as *session config options* — a list of
select and boolean entries, each with an id (`model`, `mode`, `effort`, …), a
current value and its available choices. RoomKit records them when a session
opens and follows the agent's `config_option_update` notifications:

```python
agent.session_config(room_id)
# {"mode": "auto", "model": "sonnet", "effort": "xhigh"}

agent.config_options(room_id)          # full descriptors, for a picker
# [{"id": "model", "name": "Model", "currentValue": "sonnet",
#   "options": [{"value": "opus[1m]", "name": "Opus"}, ...]}, ...]

await agent.set_config_option(room_id, "model", "opus[1m]")
# {"mode": "auto", "model": "opus[1m]", "effort": "xhigh"}
```

Both readers are empty until the room's session exists — sessions open on the
first prompt. `set_config_option()` opens one if needed (which starts the
agent process), and returns the mapping the agent reports back, so you see the
value it actually landed on: agents resolve aliases, and a change can
invalidate a sibling option.

Every change — the agent's own, or one you make — publishes an ephemeral
`CUSTOM` event so UI surfaces can follow along:

```python
{
    "type": "acp_config_options",
    "session_id": "sess-1",
    "values": {"model": "opus[1m]"},        # ids, for logic
    "labels": {"model": "Opus"},            # display names, for a status bar
    "config_options": [...],                # full descriptors
}
```

!!! warning "A slash command inside the agent is invisible here"

    Verified against `claude-agent-acp` 0.61.0: typing `/model sonnet` at a
    RoomKit prompt *works* — the Claude SDK handles the command locally,
    without invoking the model — but the adapter relays only its text output
    and sends **no** config update. The switch is real and RoomKit never
    hears about it, so `session_config()` and anything built on it go stale.
    Drive switches through `set_config_option()` when the value must stay
    observable. The Claude Code example intercepts `/model` in its
    `content_factory` and does exactly that.

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
to rebuild the accumulated Markdown document on every text delta. Set
`CONSOLE=1` to upgrade to the branded console mode (startup banner, styled
tool activity — see the [CLI Channel & Console Mode guide](cli-channel.md)). Headings,
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

The model is pinned at startup with `--model` (forwarded as
`ANTHROPIC_MODEL`, the adapter's highest-priority model source) and switched
mid-session with the example's own `/model` command, which routes through
`set_config_option()` so the console's status bar follows:

```bash
uv run python examples/acp_claude_code.py --model sonnet
```

```text
❯ /model
Model: sonnet
Available: default, opus[1m], claude-fable-5[1m], sonnet, haiku

❯ /model opus[1m]
Model: opus[1m]
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

## Two agents in one Room

`examples/acp_multi_agent.py` puts Claude Code **and** Codex in the same Room
and the same working directory, so you can have one write code and the other
review what landed on disk:

```bash
CONSOLE=1 uv run python examples/acp_multi_agent.py
```

```text
❯ @claude-code write hello.py
❯ @codex what did claude just do?
❯ /agent           # keyboard menu: pick the agent you address
```

Two settings make this work, and no routing rules at all:

```python
# An agent's own output solicits nobody it did not address itself. Set on the
# room that turned multi-agent, not on the kit: a kit-wide default would also
# silence the single-agent rooms a server hosts alongside it.
await kit.set_agent_response_policy(room_id, AgentResponsePolicy.ADDRESSED_ONLY)

# Every submission names its recipient (RFC §19.3).
await cli.run(
    kit,
    room_id=room_id,
    addressed_to=lambda _line: [addressed.agent_id],
    ...
)
```

1. **Only the addressed agent runs.** `addressed_to` names it on the event and
   `EventRouter` skips every other intelligence channel — ahead of any routing
   decision (RFC §19.4 step 0).
2. **Agents do not trigger each other.** Under `ADDRESSED_ONLY` an agent's own
   output solicits only what it addressed itself. Without that, the first
   answer would reach the second agent, whose answer would come back to the
   first, until `max_chain_depth` (5) stopped it.

### Catching up on what it missed

A non-addressed agent is skipped entirely — not asked, and not told (RFC
§19.3.2). For an `AIChannel` that costs nothing: its context is rebuilt from
the store every turn. An ACP agent keeps its history **inside its session**, so
what it was not told would be gone for good — and the failure is quiet. Asked
whether it saw the previous message, it answers yes, in good faith, about its
own session preamble.

So `ACPChannel` reads the room's timeline the moment it *is* addressed, and
prefixes what it missed to the prompt:

```text
[Room context — 2 messages you did not receive. Context only; the request follows.]
[1] Marie · sms: on part sur quoi ?
[2] claude-code: I wrote hello.py
[End of room context]

what did claude just do?
```

- **Only the gap.** What arrived since this agent's last prompt, never what its
  session already holds — an ordinary back-and-forth carries no block at all.
- **Only what it may see.** The block is filtered per reader (RFC §7.5 rule 8),
  so a message scoped away from the agents stays out of every session. Catching
  up is not a second door into the room.
- **Honest about its bound.** `room_history` (default 20) caps the block, and
  the header says so when it bites: *"the 20 most recent of 47 messages you did
  not receive"*. An agent that knows it was truncated can ask for the rest; one
  that believes it holds the whole room cannot.
- **Its own words stay out**, and a session closed and reopened starts over —
  the new one has missed everything.

```python
ACPChannel("codex", command=[...], cwd=workspace, room_history=0)  # opt out
```

`recent_events_window` follows `room_history`. The framework sizes the room tail
it loads to the largest window any bound channel declares, under a floor it keeps
for hooks (50 events) — so the default reads a tail that was loaded anyway, and
raising `room_history` past the floor grows the tail the catch-up draws on.

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

### The end of a turn is announced

An agent that owns its own turn still reports it finished. When a turn reaches
its end, the channel fires `ON_AI_RESPONSE` with the text it produced, how many
tools it called, and how long it took — the same trigger, and the same
`AIResponseEvent`, an in-process AI channel fires. Post-processing an
integrator hangs off that hook therefore runs for a conversation an agent held:

```python
@kit.hook(HookTrigger.ON_AI_RESPONSE, execution=HookExecution.ASYNC)
async def summarize(event, ctx: RoomContext) -> None:
    logger.info(
        "%s answered in %sms with %d tool calls (%s tokens)",
        event.channel_id,
        event.latency_ms,
        event.tool_calls_count,
        event.usage.get("total_tokens"),
    )
```

Two things are worth knowing about that event.

**Only a finished turn reports.** A stream closed from the outside — its
consumer cancelled, a muted binding dropping it — cancels the agent and
delivers nothing to the room, so it is not a response and fires nothing. Nor
does a turn that ended in an error.

**`usage` is the agent's own accounting, relayed unaltered.** The token
counters come off the prompt's response; the context occupancy and running
cost come off the usage notifications the session sends:

```python
{
    "input_tokens": 2, "output_tokens": 3, "total_tokens": 27369,
    "cached_read_tokens": 16997, "cached_write_tokens": 10367,
    "context_used": 27369, "context_size": 1000000,
    "cost": 0.1128, "currency": "USD",
}
```

Read `total_tokens`. A coding agent's context arrives almost entirely as cache
reads, so the numbers above are a real turn: `2` uncached input tokens against
`27369` actually accounted for. Any per-turn cost computed from `input_tokens`
alone is off by four orders of magnitude.

The ACP schema annotates those counters as running session figures ("total
input tokens across all turns"), while the reference agent fills them per
prompt — measured against it, `cached_read_tokens` is the whole prefix re-read
on that turn rather than a sum over turns. A client cannot tell the two apart
from one reading, and reinterpreting either way corrupts the figure where
nothing downstream can notice, so RoomKit does no arithmetic on them at all. If
an integrator knows which convention its agent follows, that is where the
subtraction belongs. Only `context_used`, `context_size` and `cost` are
dependably cumulative — they climb turn after turn.

An agent that reports no usage at all leaves `usage` empty rather than
inventing zeros.

### A turn never outlives its tool calls

An agent that disappears mid-tool — its process restarted, its host gone — sends
no terminal update for the call it was running. The tool-call start is already
persisted, so nothing would ever close it and the card reads as *running* on
every reload of the conversation, indefinitely.

However a turn ends, therefore, every tool it started and left unfinished is
closed first: a tool-call end with status `failed`, carrying an error that says
the turn ended before the tool reported a result. That distinction is for
whoever reads the thread later — a tool that never returned because the turn
died is not a tool that failed on its own. Cancellation goes the same way: a
stop the user asks for returns through the ordinary end of a prompt, and takes
the open tool with it. A turn whose tools all reported adds nothing.

One case stays open by construction. A response stream closed from the outside —
its consumer cancelled, a muted binding dropping it — is past the point where
anything can be added to it, so the live surfaces get their ephemeral tool-call
end and the stored row stays pending.

### What a tool result shows

ACP fixes the *envelope* — `content` blocks and a free-form `raw_output` — and
leaves the payload inside it to each agent. Two agents in one room therefore
answer in two dialects, and console mode reads both:

| The agent sends | The console shows |
|---|---|
| A `text` content block (Claude Code) | Its lines |
| A `diff` block with old/new text | Colored ± lines under the file path |
| `raw_output: {"formatted_output": …, "exit_code": N}` (Codex) | The command's output — `exit code N` when a failure printed nothing |
| A `terminal` block — a handle on live output, carrying no text | Whatever `raw_output` holds, since the block itself has none |
| An MCP `{"result": {"content": […]}, "error": …}` wrapper | The result's text, or the error |
| An `image`, `audio`, or `resource` block | The medium, named — base64 is never printed |
| Anything else | Compact JSON, capped |

Previews are capped at five lines with a `… +N lines` marker, and each line at
200 characters. Output that merely *happens* to be JSON — `cat package.json` —
keeps its own lines instead of being taken apart.

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

A handler serves **one** channel — that is what makes the injected hooks
attributable — and it can say which, so a prompt can name who is asking:

```python
async def process_tool_call(self, tool_name, tool_input, **kwargs):
    answer = await terminal_input(f"@{self.channel_id} wants {tool_name}. Allow? [y/N] ")
    ...
```

`channel_id` is empty until the channel is registered (handlers are built
first), so read it when a tool call arrives, not in `__init__`. Wiring the
same instance to a second channel logs a warning and re-attributes its tool
events — give each channel its own handler.

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

- Stable ACP protocol v1, over stdio or any transport you supply
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
