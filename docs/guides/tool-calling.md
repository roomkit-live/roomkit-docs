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

A tool's name has to suit the providers it reaches. `AITool` refuses a name
no vendor accepts: an empty one, or one with a character other than a letter,
a digit, `_`, `.`, `:` or `-`. Each provider checks its own vendor's rule when
it declares the turn's tools and raises a `ProviderError` naming the tool
before the request, rather than letting the vendor reject the turn:

| Provider | Tool names it accepts |
|---|---|
| OpenAI (its own endpoint), Anthropic | `[A-Za-z0-9_-]{1,128}` |
| Gemini | `[A-Za-z_][A-Za-z0-9_.:-]{0,127}`: a dot and a colon, no leading digit |
| Mistral | `[A-Za-z0-9_.-]+`: a dot, no colon |
| A server behind a custom URL (`base_url`, Mistral's `server_url`) | not checked: the server decides |

A realtime provider checks the same way when it declares a session's tools, at
connection and at every reconfiguration. Where the endpoint hands the tools to
another model, the rule is that model's vendor's (measured 2026-10-02):

| Realtime provider | Tool names it accepts |
|---|---|
| OpenAI Realtime and GPT-Live with `HostedReasoning`, each on its own endpoint | `[A-Za-z0-9_-]{1,128}` |
| Deepgram | its think provider's: `open_ai` and `anthropic` as above, `google` as Gemini; any other, or a `think_endpoint`, not checked |
| Gemini Live | as Gemini |
| xAI | not checked: it accepts any name |
| GPT-Live with `IntegratorReasoning`, ElevenLabs | not checked here: the reasoning backend's provider, or the agent's own configuration, decides |

So an MCP tool named `files.read` works on Gemini and Mistral and is refused
up front on OpenAI and Anthropic. A tool without parameters (`parameters={}`
or none) is declared as an object with no properties, which every provider
accepts, and a schema whose root has no `type` is declared an object's:
Anthropic and OpenAI refuse it untyped.

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
{"action": "list_columns", "board_id": "board-1"}
```

The schema is closed, so the argument gate would refuse `board_id` and the turn
would be spent on an error the model can only fix by re-issuing the call.
RoomKit folds the call back into shape instead, before validation, on both the
AI and realtime voice channels — the handler receives
`{"action": "list_columns", "params": {"board_id": "board-1"}}` and the round
does real work. Each fold is logged at INFO with the tool and the model id, so
the frequency of the case stays measurable per model.

The repair is deliberately narrow. It applies only when the schema closed
itself, declares a `params` property of type `object` **that declares no
properties of its own**, and carries at least one undeclared root key — and only
when `params` is absent or empty:

| Call | Outcome |
|------|---------|
| `{"action": "x", "board_id": "1"}` | folded into `params` |
| `{"action": "x", "params": {}, "board_id": "1"}` | folded into `params` |
| `{"action": "x", "params": {"a": 1}, "board_id": "1"}` | **refused** — both forms at once is ambiguous; the error says to pass every argument inside `params` |
| `{"city": "Laval", "units": "metric"}` on a flat tool | **refused** — no container to fold into, so an unknown argument stays an error |
| `{"titel": "Q3"}` on a tool whose `params` declares its own properties | **refused** by name — that `params` is an options object, not a hub container |
| any call against a schema without `additionalProperties: false` | untouched — undeclared root keys are already legal there |

The shape condition matters as much as the name, because `params` is an ordinary
name for an ordinary options object:

```python
"properties": {
    "title": {"type": "string"},
    "params": {"type": "object", "properties": {"width": {"type": "integer"}}},
}
```

A hub container **cannot** declare its properties — its shape varies with
`action` — so a declared shape means the tool is not a hub. Folding into it
would move a misspelt *root* property (`titel`) inside the container, where the
gate cannot see it: validation does not recurse into nested objects. The call
would then reach the tool with a bogus key and no title. Left alone, the same
call is refused as `unknown argument 'titel' (this tool accepts: params, title)`
— which is the one thing the model can act on.

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

from roomkit import UnservedToolCallError


async def my_handler(name: str, arguments: dict) -> str:
    if name == "get_weather":
        city = arguments["city"]
        # Call your weather API
        return json.dumps({"temp": 22, "condition": "sunny"})
    if name == "search":
        query = arguments["query"]
        # Search your knowledge base
        return json.dumps({"results": ["result1", "result2"]})
    raise UnservedToolCallError(f"{name} is not mine")  # not this handler's tool


ai = AIChannel("ai", provider=provider, tool_handler=my_handler)
```

When both `tools` and `tool_handler` are provided, the channel merges them — Tool object handlers are tried first, then the explicit `tool_handler`.

`channel.tool_handler` is this handler, the host's: reading it returns it, and
assigning it replaces it alone. The tools the channel serves itself (skills,
Tool Search, `read_stored_result`, the planner, the sandbox) and the ones
orchestration sets up (a handoff, a delegation, a strategy's tools) are served
before it, whatever it is.

### One tool per name

A name is served by one tool in a room, so a tool given under a name the
channel already has is refused when it is given (RFC §21.1):

- two tools of the host under one name raise `ValueError` when they are given
  (an `AIChannel` or a `RealtimeVoiceChannel` at construction, a realtime
  `configure(tools=)`, a conference's `ConferenceRealtimeConfig`): composing two
  MCP servers that both expose `search` would otherwise declare one server's
  schema for a call the other serves;
- `setup_handoff`, `setup_delegation` or a strategy setting a tool up under a
  name one of the host's tools carries raise `roomkit.ToolNameCollisionError` (a
  `ValueError`): rename one of them. The same strategy installed again in a room
  replaces its own tools, and an install that sets up several tools sets up all
  of them or none.

A tool that arrives with the turn (binding metadata, a `config_provider`, a
`BEFORE_AI_GENERATION` hook) comes too late to be refused: under a name the
channel or orchestration serves it is left out, with a warning, and a name it
gives twice is declared once, with the later definition.

A handler answers with text, or with a list of content parts (text and images)
for a multimodal result. Anything else it returns (a dict, a list of values, a
number, `None`) reaches the model as its JSON serialization, the same on
`AIChannel` and `RealtimeVoiceChannel`: `{"ok": True}` reads as `{"ok": true}`
and `None` as `null`. A result an `ON_TOOL_CALL` hook supplies in its place is
read the same way.

The pre-execution gates — the declared-catalogue check, argument validation
against the declared schema, and the `BEFORE_TOOL_USE` hook — are a property of
the channel, not of the handler. They run before the call is routed, so a tool
served by an `ON_TOOL_CALL` hook on a `RealtimeVoiceChannel` with no
`tool_handler` is gated exactly like one served by a handler.

A declared tool that no handler serves reaches the `ON_TOOL_CALL` sync hooks
with `result=None`: a hook may serve it by supplying the result (`HookResult`
with `metadata={"result": ...}`). If none does, the model reads
`{"error": "No handler for tool <name>"}` and the call is reported once, as
failed. The channel's own outcomes are refusals too: a repeat of the same call
with the same arguments that the channel stops, and a tool outside the turn's
toolset. `HumanInputToolHandler` refuses a request nobody answered in time, or
one the human rejected.

A call one of those gates refuses never reaches the handler, and a handler that
raises never returns — but both are still reported. They fire `ON_TOOL_CALL`
with `is_error=True`, on the async observers only: an audit hook sees the
refusal, while a hook that could have served the call does not, so a denial
prevents the side effect instead of hiding it. Read the outcome from `is_error`
rather than from the result text, which is written for the model.

On a realtime channel the model can also abandon a call it issued: Gemini Live
sends `tool_call_cancellation` when the caller interrupts while the tool runs.
The channel cancels the handler (it sees `asyncio.CancelledError` at its next
`await`), sends nothing back, and fires the observers with `cancelled=True`
beside `is_error=True`. The provider-level callback is described in the
[realtime providers guide](realtime-voice-providers.md#background-tool-calls).
An AI channel reports the same way every call its turn announced and nothing
else reported, whatever cut it: a stop while the calls were announced, a
transport that stopped reading (a voice barge-in), the turn cancelled in the
gate, in the handler or while `ON_TOOL_CALL` judged the call. Each is stored
`cancelled` and reaches the observers once, with `cancelled=True`. A call
whose outcome the model already read (one the provider ran, one an external
handler decided, a realtime Tool Search result already sent) keeps that
outcome when a cut interrupts its own report: the observers hear it once,
with that outcome, never a second time cancelled.

A handler that declines a call it owns raises `ToolRefusedError`. A refusal
returned as a body reads as work that was done, and any other exception reads
as `{"error": "Tool '<name>' failed (<ExceptionClass>)"}` on every channel:
its message never reaches the model, since it can hold anything the failing
code held (a connection string with its password), and goes to the log and to
`ON_TOOL_CALL` observers as `event.error_detail`. Raising `ToolRefusedError`
keeps both: the call is marked `is_error`, the observers fire, and the message
reaches the model verbatim.
`MCPToolProvider.as_tool_handler()` raises it when the server refuses a call.
A tool that is not the handler's own is a different case: raise
`UnservedToolCallError`, so a composed handler passes the call on (see below)
and the channel reads the call as served by nothing, on every path (RFC §21.4).
The `{"error": "Unknown tool: ..."}` answer an earlier convention returned is
still read the same way, as text or as a mapping.

```python
from roomkit import ToolRefusedError


async def my_handler(name: str, arguments: dict) -> str:
    if name == "delete_account" and not arguments.get("confirmed"):
        raise ToolRefusedError("Refused: deleting an account needs confirmed=true.")
    ...
```

!!! tip
    Raise `UnservedToolCallError` for a tool that is not yours. This is what tool handler composition reads (see below), and what lets an `ON_TOOL_CALL` hook still serve the call.

## What a Handler Knows About the Call

The handler protocol is `(name, arguments) -> str` — no room, no speaker, no
toolset. That omission is deliberate: an `AIChannel` object is registered once
per `channel_id` and shared by every room it serves, so anything a handler
closed over when it was built describes whoever attached it, not the turn now
running. These accessors read the current turn from a contextvar instead:

| Accessor | Answers |
|----------|---------|
| `current_tool_room_id()` | Which room this turn belongs to |
| `current_tool_room()` | The turn's `Room` itself, the object `RoomContext.room` holds for the same turn: read its `organization_id`, `metadata` or `status` without a store round trip. It is the room as the store loaded it when the turn began, shared with the whole turn: a patch written to the store mid-turn is not in it, and the object itself must not be mutated (room changes go through the store); on a realtime tool call, which runs no turn, the room as loaded for that call |
| `current_tool_actor_id()` | Whose turn it is — the participant id of the event that woke the channel |
| `current_tool_allowed_names()` | Every tool name the turn resolved, and every tool a round declared beyond it (`read_stored_result`, `plan_tasks`), nothing withdrawn, that its tool policy admits, so a call is validated against the live toolset rather than an attach-time snapshot (a tool a skill keeps closed stays in); on a realtime tool call, every tool the session declares that its policy admits (`None` when it declares no catalogue: no list, the gate still judges each call) |
| `current_tool_call()` | The per-call context: the call's id, its channel, and the `structured_content` reverse channel the handler may fill |
| `current_response_metadata()` | The turn's response-metadata record — what the reply's MESSAGE events will carry (see below) |

Contextvars propagate down the async call chain, so they work at any depth
without a signature change. The realtime voice channel and a conference
install the same context around each tool call they serve (the session's room
and participant as the turn's room and actor; a conference's mix names no
participant, so its actor is `None`), so one handler works on every path for
the room id, the `Room`, `current_tool_call()` and
`current_tool_allowed_names()` (there, every tool the session declares that
its policy admits: its catalogue, what orchestration set up, the channel's
own; `None` for a session that declares no catalogue: no list, its gate still
judges each call);
`current_response_metadata()` returns `None` there (no turn merges a record on
that path, so the `if record is not None` guard below skips a write nothing
would carry). Each returns `None` outside a
tool call (a direct call) — keep your own fallback there.

```python
from roomkit.tools import current_tool_actor_id, current_tool_room, current_tool_room_id


async def my_handler(name: str, arguments: dict) -> str:
    room_id = current_tool_room_id()
    actor_id = current_tool_actor_id()
    room = current_tool_room()  # the same object the turn's RoomContext.room holds
    tenant = room.metadata.get("tenant") if room is not None else None
    ...
```

### What a Handler Can Tell the Turn

A handler's return value is what the model reads. Two things it learns belong
to the turn instead, and both travel through the per-call context:

- **`current_response_metadata()`** is the turn's one `ResponseMetadata` record
  (`roomkit.models`): a dict-like mapping created with the turn and merged into
  every MESSAGE event the turn produces, as it stands when each event is
  created. A memory provider writes it while the context is built, a
  `BEFORE_AI_GENERATION` hook writes `event.ai_context.response_metadata`, and a
  tool handler or a `BEFORE_TOOL_USE` hook writes here: all of them reach the
  same object, the one `InboundResult.response_metadata` hands the caller,
  whether the turn answered or failed. A document the tool read mid-loop can
  therefore be named as a source of the reply, and a host can count the calls a
  turn started.
- **`current_tool_call().structured_content`** is the structured copy of an MCP
  result that the tool-call events persist verbatim for UI surfaces. A handler
  that rewrites the text before the model reads it (a provider's private
  address turned into a relay link, say) rewrites the copy here too. The copy
  reaches `ON_TOOL_CALL` the same way on an AI channel, a realtime session and
  a conference: a SYNC hook sees it and may replace or clear it, and the
  observers receive what the chain left.

```python
from roomkit.tools import current_response_metadata


async def my_handler(name: str, arguments: dict) -> str:
    result = await call_my_tool(name, arguments)
    record = current_response_metadata()
    if record is not None:
        record.setdefault("cited", []).append({"tool": name, "id": result["id"]})
    return result["text"]
```

Segments streamed before a tool round carry what was known then; the answer,
persisted after it, carries what the handler wrote. `None` outside a turn.

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

!!! note "What the policy covers"
    The policy governs every tool the channel offers, including the ones it injects itself: sandbox commands (`sandbox_*`), `run_skill_script` and `plan_tasks` must be allowed like any host tool. Six tools only read or unlock and are never filtered when the channel serves them itself: `activate_skill`, `read_skill_reference`, `read_stored_result`, `find_tools`, `list_tools`, and `call_tool`, which only carries a call on a fixed-declaration realtime provider: the policy judges the tool it names. A provider's native tool without a name (`{"google_search": {}}`) is never filtered nor hidden by Tool Search: a policy names the tools it governs. A tool of yours or of an MCP server under one of these names is filtered like any other; better, don't use them: a tool passed to the channel's `tools=` under a name the channel serves raises `ValueError` at construction (pass `tool_search=False` to free `find_tools` and `list_tools`), and one that arrives later (a binding's tools, orchestration) is not declared, with a warning. No name is declared twice. The same rule decides what the model is offered and what it may call, and Tool Search's `find_tools` / `list_tools` never name a tool the policy denies or a skill gates.

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

A tool the server lists under a name no provider accepts (see
[the AITool model](#aitool-model)) is skipped with a warning; the server's
other tools stay available.

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
# local_handler is tried first; if it declines the call, mcp_handler is tried
```

A handler declines a call by raising `UnservedToolCallError`, or with the earlier `{"error": "Unknown tool: ..."}` answer. Any other response (including other errors) is treated as a valid result and returned immediately. The last handler's answer is the composition's, a decline included: the channel then reads the call as served by nothing.

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

The streaming loop emits `StreamEvent` objects: `StreamTextDelta`, `StreamThinkingDelta`, `StreamToolCallDelta`, `StreamToolCall`, and `StreamDone`. Tools are executed concurrently via `asyncio.gather()`.

`StreamToolCallDelta` carries one fragment of a call's arguments as the model composes them, so a long composition is observable while it happens; `StreamToolCall` still follows and remains the unit of execution and persistence.

## Tool Call Events

AIChannel automatically publishes ephemeral `TOOL_CALL_DELTA`, `TOOL_CALL_START` and `TOOL_CALL_END` events that you can subscribe to:

```python
from __future__ import annotations

from roomkit.realtime import EphemeralEvent, EphemeralEventType


async def on_tool_event(event: EphemeralEvent) -> None:
    if event.type == EphemeralEventType.TOOL_CALL_DELTA:
        # The call being composed: its name and how far along, never the
        # argument content. See the realtime-features guide.
        call = event.data["tool_calls"][0]
        print(f"Composing {call['name']}: {call['arguments_chars']} chars")
    elif event.type == EphemeralEventType.TOOL_CALL_START:
        tools = event.data["tool_calls"]
        print(f"Calling: {[t['name'] for t in tools]}")
    elif event.type == EphemeralEventType.TOOL_CALL_END:
        # status is "completed" or "failed": a refused, failed, blocked,
        # unserved or cancelled call ends "failed".
        failed = [t["name"] for t in event.data["tool_calls"] if t["status"] == "failed"]
        print(f"Completed in {event.data.get('duration_ms')}ms, failed: {failed}")


sub_id = await kit.subscribe_room("room-1", on_tool_event)
```

The streaming-only `TOOL_CALL_DELTA` reports the tool name and cumulative
argument size, never the argument content. An empty `tool_calls` terminal frame
closes every composition attempt, including cancellation, provider failure,
retry, or fallback. Counts restart when a retry begins; the complete arguments
still arrive once in `TOOL_CALL_START`.

## Cross-turn tool memory

The AI context rebuilt for each turn contains MESSAGE events only — tool-call
events are filtered out (providers track tool context *within* a turn, not
across turns). Left alone, the model would lose all trace of the tools it
invoked from one turn to the next: it couldn't tell which tool or source it
already used, and under Tool Search it couldn't re-call a tool it used a moment
ago because the catalogue is re-hidden every turn.

`AIChannel` closes both gaps automatically with a per-room, in-memory record of
tool usage — no configuration needed:

- **A "what you did" digest** — recent tool calls (name + arguments) ride
  the turn's input, after the user's words and marked as the runtime's notes,
  so the model knows what it already did and what it got. They are not in the
  system prompt: it changes after every turn that calls a tool, and a system
  prompt that changes invalidates a provider's cache of the whole history
  (RFC §6.4). The room's plan (`enable_planning`) travels the same way. The three most recent keep their result, up to 6,000 characters
  each and never more than the eviction threshold (`evict_threshold_tokens`)
  lets through: a follow-up question ("and the fifteenth board?") is answered
  from the data instead of invented. Each result sits in a `<tool_result>`
  block the model is told to read as data, never as instructions, since it
  comes from a tool and not from the prompt's author. A longer result is cut
  and marked so, telling the model to call the tool again rather than guess;
  older calls shrink to a short preview, set apart as data too. In a live conversation
  the result kept is what the tool returned, even when eviction gave the model
  a placeholder for it. Bounded by recent *calls*.
- **Sticky re-exposure** — the distinct tool names called recently are
  re-revealed each turn, so a tool used once stays callable even while Tool
  Search hides the rest of the catalogue. Bounded by recent distinct *tools* —
  the conversation's working set — since this is the part that costs full tool
  schemas.

On a provider that can hold a tool declared but unseen (Anthropic, `AIProvider.supports_deferred_tools`), the hidden catalogue is declared that way (`defer_loading`) from the first round, and `find_tools` makes its matches callable by reference: the tool list does not change within the turn, so the provider's prompt cache holds across the reveal (RFC §6.4). A skill's gated tools are held the same way until `activate_skill` opens them. The declaration holds from one turn to the next as well: the tools a room's first turn showed stay the ones shown, and a tool an earlier turn opened (revealed, used, unlocked) stays held. Its reference rode a tool result the next turn's history does not replay, so the turn starts with a short exchange of its own, placed after the history and before the input: a `find_tools` call (or, without Tool Search, an `activate_skill` call for each active skill that gates one) whose result references the tools it reopens. The exchange is the turn's context only, never stored, delivered or counted as a call, a standalone instruction neither reads nor keeps the room's declaration, a `fallback_provider` that cannot hold tools gets them declared instead of the exchange, and the history stays in the cache where a new tool list would rewrite it all. Anthropic behind a `base_url` (a proxy or a gateway) holds nothing, since what it forwards to may not accept the form, and a `fallback_provider` that cannot hold a tool receives the tools the turn made callable, declared plainly.

Tool Search never hides a tool orchestration injected: a handoff
(`handoff_conversation`), a delegation (`delegate_task`, a supervisor's tools)
or a delegation's `submit_result` stays declared as a pinned tool does, is not
named by `find_tools`, and does not count toward the catalogue that switches
Tool Search on. The agent is told to call these tools, not to go and find them.

A `BEFORE_AI_GENERATION` hook sees the whole catalogue under Tool Search,
deferred tools included, so it can withdraw one: `find_tools` and `list_tools`
do not name it and a call to it is refused. An `AFTER_TOOL_ROUND` hook, between
two rounds, withdraws with the same guarantees, and its event names what
`BEFORE_AI_GENERATION` is shown (`tools`): the whole catalogue the policy and
skill gating let the turn reach, nothing withdrawn. A tool the hook adds is declared at
every round of the turn, as a pinned tool is, and `find_tools` does not name it.

The record is scoped per room and kept in memory; after a process restart it is
rebuilt once per room from the channel's own persisted tool-call events, so a
conversation that outlives its channel object keeps its tool memory. Another
agent's calls in the room are not part of it, whatever that agent let it see,
and each call keeps the arguments of its own turn. Those
events hold what the model was given: a result that had been evicted comes back
as a short preview, not as the stored id.

## Tool Loop Configuration

```python
ai = AIChannel(
    "ai",
    provider=provider,
    tools=[my_tool],
    max_tool_rounds=50,              # Max iterations (default: 50)
    tool_loop_timeout_seconds=300,   # Hard timeout in seconds (default: 300)
    tool_timeout_seconds=30,         # One call's bound in seconds (default: 30)
    tool_loop_warn_after=25,         # Soft warning threshold (default: 25)
    turn_budget_usd=0.05,            # What a turn may cost (default: no budget)
)
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_tool_rounds` | `50` | Maximum tool loop iterations before forced stop |
| `tool_loop_timeout_seconds` | `300.0` | Hard timeout for entire loop. `None` disables |
| `tool_loop_warn_after` | `25` | Log warning at this round count |
| `tool_timeout_seconds` | `30.0` | How long one tool call may take. `None` disables |
| `tool_timeouts` | `None` | A bound per tool name, above the default (`None` for no bound) |
| `turn_budget_tokens` | `None` | Billed tokens a turn may spend, cache included |
| `turn_budget_usd` | `None` | What a turn may cost at the model's catalogue price |

A turn can also be capped by what it spends: `turn_budget_tokens` counts every token the provider bills for the turn (input, cache reads and writes, output), `turn_budget_usd` prices each generation at the model's catalogue rate. At the first round boundary where the turn has reached either, the loop ends `budget_exceeded`: the calls that round asked for do not run and no further generation is asked for, so the turn overshoots by one generation at most. Both are off by default and can be set per room (binding metadata) or per turn (`AIChannelTurnConfig`). A budget that is not a positive number, or a cost budget on a model with no catalogue price, raises `ValueError`: on the channel when it is built, otherwise in the turn that reads it. A generation the `fallback_provider` serves is priced at the primary provider's rate; a fallback priced otherwise is logged once (RFC §6.4).

!!! note
    A tool result over `evict_threshold_tokens` (5,000 by default) is stored and replaced by a preview the model can page back with `read_stored_result`, or search with its `query` for the lines that contain one line of text: a search reads every line of the result whole, so no match means the text is absent (RFC §21.5).

When the provider refuses a round's context as too long, the channel compacts it once and replays the round. The turn's input and its notes (plan, tools already used, speakers) stay whole. When the input falls in the older half of the messages, the history before it is summarized and the long results of the turn's older tool rounds are stored like an evicted result, a short preview in their place, so the model can page them back with `read_stored_result`; a skill's instructions and a page already read back stay whole. Otherwise the older half is summarized. Every call keeps its result, a summary joins the user message that follows it instead of forming a second one in a row (RFC §6.4), and a context with nothing left to shorten before the input fails the round rather than cutting the input.

### Tool call timeout

The loop's timeout is read between rounds, so it cannot stop a handler that never answers. Every call has its own bound for that (RFC §21.6): past `tool_timeout_seconds`, the handler is cancelled and the call fails like one whose handler raised, the model reading `{"error": "Tool 'x' failed (ToolTimeoutError)"}` and the observers the detail, and the turn goes on.

```python
ai = AIChannel(
    "ai",
    provider=provider,
    tool_handler=handler,
    tool_timeout_seconds=30,                         # every call (default: 30)
    tool_timeouts={"export_report": 120, "train": None},  # a tool's own bound
)
```

The same two settings exist on `RealtimeVoiceChannel` and `ConferenceRealtimeConfig`, with 10 s by default: a person waits in silence for the answer. A cascade voice agent answers through an `AIChannel` and takes its 30 s unless you lower it. A tool that keeps a bound of its own is not subject to the default: one that waits on another agent or a person by design (a delegation, a supervisor's or a loop's strategy tool, a `HumanInputToolHandler`'s tools) and `sandbox_bash`, whose `timeout` argument your `SandboxExecutor` enforces. A bound set in `tool_timeouts` still applies to them. A pipeline agent's own tools are yours and bounded like any other. A `TimeoutError` your handler raises itself is its own failure, not an expired bound. A bound that is not a positive number, nor `None`, raises `ValueError` when the channel or the config is built.

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
