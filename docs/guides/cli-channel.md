# CLI Channel & Console Mode

`CLIChannel` is an interactive terminal transport: it reads lines from stdin,
feeds them into a room as inbound messages, and prints agent responses to
stdout. It is the fastest way to talk to any RoomKit setup — an `AIChannel`,
an orchestration (swarm, supervisor, pipeline), or an external coding agent
via `ACPChannel` — without a web frontend.

```python
from roomkit import ChannelCategory, CLIChannel, RoomKit
from roomkit.channels.ai import AIChannel

kit = RoomKit()
cli = CLIChannel("cli")
ai = AIChannel("ai-assistant", provider=my_provider)

kit.register_channel(cli)
kit.register_channel(ai)

await kit.create_room(room_id="demo")
await kit.attach_channel("demo", "cli")
await kit.attach_channel("demo", "ai-assistant", category=ChannelCategory.INTELLIGENCE)

await cli.run(kit, room_id="demo")
```

`run()` blocks on the terminal until the user types `quit`/`exit` (or sends
Ctrl+D / Ctrl+C). Everything else — routing, hooks, broadcast — is the normal
inbound pipeline.

## Rendering modes

| Mode | Enable | Output | Extra |
|------|--------|--------|-------|
| Plain (default) | — | ANSI-colored text, streamed token by token | none |
| Markdown | `markdown=True` | Progressively re-rendered Markdown (headings, lists, code blocks) | `console` |
| Console | `console=True` | Markdown **plus** startup banner, brand palette, styled tool activity | `console` |

Both rich modes need the `console` extra:

```bash
pip install "roomkit[console]"
```

`console=True` subsumes `markdown=True`; passing both is allowed and behaves
like console mode.

## Console mode

Console mode gives the CLI the feel of a modern agent CLI (Claude Code,
Codex): a branded startup banner, an input bar pinned to the bottom of the
terminal, progressive Markdown, and tool-call activity rendered as it
happens.

```python
cli = CLIChannel("cli", console=True, show_thinking=True)
```

At startup the banner shows the RoomKit logo and version, the AI model(s)
found in the room, the room id, and the attached channels:

```text
██ ██  RoomKit v0.40.0
██ ██  ◆ claude-sonnet-5 (anthropic)
       Room: demo  ·  Channels: cli, ai-assistant (intelligence)
       Type 'quit' to exit.

❯
```

- **Models are discovered, not configured.** For every binding attached with
  `ChannelCategory.INTELLIGENCE`, the banner reads the channel's public
  `provider` and its `model_name` (`AIChannel.provider` is public API).
  Rooms with several AI channels get one line per model; orchestration-only
  rooms (agents attached by the orchestrator) and agent channels without a
  model name (e.g. `ACPChannel`) simply show what is known.
- **`welcome` becomes the notes line.** The `welcome` string passed to
  `run()` renders dim under the banner instead of being printed raw — keep
  only what the banner cannot know (usage hints, env toggles).
- **The prompt becomes `❯`** in the brand color, unless you pass an explicit
  `prompt=`.

A turn opens with the agent's handle — quiet and dim, it names who is
speaking without shouting — and the prose that follows leads with `●`, so
the answer is visually one block whatever it contains. Thinking (when
`show_thinking=True`) renders dim, tool calls render inline with their
results — output previews for commands, colored ± diffs for file edits,
capped at a few lines — and the turn closes with what the wait cost:

```text
@claude code
💭 The user wants a hello world in C.

⏺ Write
  ⎿ ✓ Write /tmp/hello.c · 3.0s
    /tmp/hello.c
    +#include <stdio.h>
    +int main(void) {
    … +3 lines

⏺ Terminal
  ⎿ ✓ cc /tmp/hello.c -o /tmp/hello && /tmp/hello · 5.2s
    Hello, world!

● Compiled and ran — output `Hello, world!`. The program prints the
  greeting and exits cleanly.
  ⎿ took 9.4s · 2 tools
```

A speaker with a participant behind them is named, with the kind of channel
they reach you through — `@marie · sms`. Two colleagues texting into one room
are told apart; an agent, having no participant, keeps its channel-derived
handle.

The marker leads each stretch of prose, so an answer resuming after a tool
round starts a fresh `●`; continuation lines align under the text, not under
the marker. A tool start renders without parentheses when its arguments are
not yet known (ACP agents enrich the tool title while it runs); failures
render `⎿ ✗ tool failed` with the error text as the preview.

A tool result is *read*, not printed: agents wrap their output differently —
ACP diff and text blocks, MCP `content` payloads, Codex's
`{"formatted_output": …, "exit_code": N}` JSON — and each wrapper is unwrapped
down to the text a reader wants. Media (`image`, `audio`, `resource`) is named
rather than dumped as base64, a command that failed silently reports its exit
code, and a payload nobody recognises still renders as compact JSON. See
[what a tool result shows](acp-channel.md#what-a-tool-result-shows) for the
full mapping.

### The pinned input bar

On a real terminal, `run()` starts an interactive shell instead of the plain
read-print loop. The input zone is pinned at the bottom of the screen —
framed by two full-width rules, with a status toolbar under it — and the
conversation scrolls above in normal scrollback. Submitted lines are echoed
into the transcript with a full-width tinted background and a muted chevron,
so your messages stand apart from agent output:

```text
❯ summarize the last meeting          ← echoed, tinted background
@assistant
● Here is the summary you asked for — …
  ⎿ took 4.1s

──────────────────────────────────────────────
❯ and now compare it with│
──────────────────────────────────────────────
 demo · Sonnet · ⠹ Claude Code working 32s · Edit · 12.3k ctx (1 queued)
```

- **Type while the agent streams.** The bar never blocks. Each submitted
  line queues, and queued messages process strictly one at a time — the
  toolbar shows `(n queued)` until the queue drains.

### The status bar: who is working

The toolbar reads `room · model · status`. While a turn is in flight the
status spins and names the agent producing it, how long the wait has run,
and what it is doing right now (`thinking`, a tool name, `responding`);
context usage joins in when the agent reports it:

```text
 demo · idle                                        ← nothing in flight
 demo · ⠙ working 2s                                ← submitted, still routing
 demo · Sonnet · ⠹ Claude Code working 32s · Edit   ← an agent is streaming
 demo · ⠴ 2 agents working 41s · Planner, Coder     ← orchestration
```

Activity is tracked **per source channel**, not as one global flag, so a
room running several intelligence channels names them all; the oldest turn
owns the clock, because that is the wait actually being lived. The spinner
runs only while work is in flight — an idle console never repaints.

The model shown is what each agent reports for itself, which replaces the
banner's startup value as soon as a session exists (an ACP agent has no
model to report until then). Several distinct models in one room collapse to
`n models`.
- **Rendering granularity.** Under the bar the response flushes per
  completed Markdown block (paragraph, list, fenced code block — fences are
  never split mid-block). In classic and non-TTY modes streaming stays
  token-level.
- **Interrupting.** **Esc** stops the turn in flight and nothing else. What
  the agent had already said stays — it is on your screen, so it is in the
  timeline too, stored with `metadata["cancelled"] = True`. The queue keeps
  draining, so a message typed behind the interrupted turn still runs.
  Cancelling is not an error: no `ON_ERROR` fires.
- **Exiting.** `quit` / `exit` / `q`, Ctrl+D, or Ctrl+C end the session — a
  different intention from Esc, hence a different key. The in-flight turn is
  cancelled and pending queued messages are dropped.

### Mid-turn terminal prompts (`terminal_input`)

Code that must ask the user a question in the middle of a turn — a tool
permission prompt, a confirmation — cannot call `input()` while the bar owns
the terminal. Use the public helper instead:

```python
from roomkit.console import terminal_input

answer = await terminal_input("Allow once? [y/N] ")
```

While the shell is active the bar is suspended for the read (the question
and answer land in the normal scrollback); without a shell it behaves as a
plain threaded `input()`. The ACP example's `TerminalPermissionHandler` uses
it.

### Picking from a list (`terminal_select`)

The other question an application asks — *which one?* — gets a keyboard
menu:

```python
from roomkit.console import terminal_select

chosen = await terminal_select(
    [("claude-code", "@claude-code · opus"), ("codex", "@codex · gpt-5.6-sol")],
    title="Address which agent?",
    default="claude-code",
)
```

```text
Address which agent?
❯ @claude-code · opus
  @codex · gpt-5.6-sol
  ↑/↓ to move · enter to choose · esc to cancel
```

It renders where the bar sits, erases itself once answered, and returns the
chosen **value** — or `None` when the user cancels (Esc, Ctrl-C). Options
are `(value, label)` pairs, or bare strings used as both. Without a shell it
degrades to a numbered list read from stdin, so piped runs still answer.

The menu works because only one reader touches the terminal at a time. Call
it from a place the loop awaits — a local command is exactly that.

### Naming who a submission asks (`addressed_to=`)

In a room with several agents, each submission can name its recipients:

```python
await cli.run(
    kit,
    room_id=room_id,
    addressed_to=lambda line: [focused_agent],   # channel ids, or None
)
```

The hook receives the submitted line and returns channel ids — or `None` to
leave the message unaddressed. It runs **after** `content_factory`, so a line
like `@codex review it` that switched the focus is already reflected when the
recipient is named.

RoomKit takes the decision, never the syntax: `@codex`, `/agent codex`, a
keyboard picker, a payload carrying its own mentions — parse it at the edge
and hand over ids. See
[Addressing](orchestration.md#addressing-naming-who-is-asked).

### Scoping a submission (`visibility=`)

Addressing says who is *asked*; visibility says who may *see*. In a room that
holds a colleague as well as agents, you need both:

```python
await cli.run(
    kit,
    room_id=room_id,
    visibility=lambda line: ["you", "sms"] if line.startswith("/dm ") else None,
)
```

Return a keyword (`"transport"`), a sequence of channel ids, or `None` for no
restriction — the default, where everything you type is visible to the whole
room. **Prefer the sequence**: `"transport,codex"` reads like "the transports
plus codex" and means neither, because the comma form matches channel ids
only. Handing ids never writes that sentence.

It sets both the message's `visibility` and its `response_visibility`, so a
private question does not publish the answer it draws — the scope covers the
whole turn, tool activity included. `examples/sms_and_agents.py` puts a
colleague on SMS in an agent's room and uses it for `/dm`.

### Your own status segment (`status_extra=`)

The bar knows the room, the models and what is running. What it cannot know
is application context — which agent your next line will reach, which mode
you are in. `status_extra` contributes one segment, asked fresh on every
render:

```python
await cli.run(
    kit,
    room_id=room_id,
    status_extra=lambda: f"→ @{focused_agent}",
)
```

```text
 coding-agents · gpt-5.6-sol · → @codex · ⠋ Codex working 12s · Shell
```

It sits between the model and the live status. Return `None` to show
nothing. A segment that raises is skipped and logged — rendering is not a
place your code can crash the console. Console mode on a real terminal only;
the classic loop has no bar.

### Local commands (`commands=`)

Lines that should never reach the room — `/model`, `/agents`, `:q` — are
declared as commands, keyed by their first word:

```python
async def switch_model(argument: str) -> None:
    ...

await cli.run(
    kit,
    room_id=room_id,
    commands={"/model": switch_model, "/agents": pick_agent},
)
```

A matching line skips `content_factory` and the room entirely; the handler
is **awaited by the loop**, in submission order, with the rest of the line
as its argument. Two consequences worth relying on:

- A handler may prompt (`terminal_input`, `terminal_select`) without racing
  the loop for stdin — the loop is not reading while it awaits.
- A command typed behind a message runs **after** that message's turn, never
  inside it. `/model sonnet` cannot land halfway through the answer it would
  have changed.

Under the pinned bar, commands ride the same submission queue as messages
(so the bar is up and can be suspended for a prompt); in the classic loop
they run between reads. The prefix is yours — RoomKit matches the first word
and nothing else.

### Inline, not a dashboard

Console mode is **not** the voice dashboard. `RoomKitConsole` (voice) takes
over the whole terminal with an alternate-screen layout — the right choice
for telemetry (audio meters, VAD state, logs). A chat transcript is the
opposite: console mode renders inline in the normal scrollback, so history
stays scrollable and copy-pastable after the session, exactly like other
agent CLIs.

### Non-TTY behavior

When stdin or stdout is not a terminal (pipes, CI), the pinned bar cannot
run: `run()` falls back to the sequential loop, color auto-detection turns
styling off, and the live renderer degrades to sequential prints. The banner
still prints — useful in CI logs. Force colors on or off with
`use_color=True/False`.

Known limitation: the render width is captured when the shell starts;
resizing the terminal mid-session does not reflow subsequent blocks.

## The `CONSOLE=1` convention in examples

RoomKit examples follow one convention for both console surfaces: setting
`CONSOLE=1` enables the rich rendering, and its absence keeps the plain
default. Voice examples call `shared.setup_console(kit)` (the dashboard);
CLI examples pass the flag to the channel:

```python
from shared import console_enabled

cli = CLIChannel("cli", console=console_enabled())
```

Try it:

```bash
CONSOLE=1 uv run python examples/ollama_ai.py                  # AI model banner
CONSOLE=1 uv run python examples/orchestration_swarm_cli.py    # orchestration room
CONSOLE=1 uv run python examples/acp_claude_code.py            # external coding agent
```

## Options reference

| Option | Default | Purpose |
|--------|---------|---------|
| `prompt` | `"You: "` (`"❯ "` in console mode) | Input prompt |
| `user_color` / `agent_color` | yellow / cyan | ANSI colors (plain mode) |
| `use_color` | auto-detected | Force colors on/off |
| `agent_label` | derived from channel id | Maps `channel_id` to a display name |
| `show_thinking` | `False` | Render model reasoning above answers |
| `markdown` | `False` | Progressive Markdown rendering |
| `console` | `False` | Branded console mode (subsumes `markdown`) |

`run()` accepts `sender_id` (a room **Participant ID**, not an address — see
[Identity Resolution](identity-resolution.md)), `welcome`, `content_factory`
(map a raw input line to custom content, or return `None` to swallow a line)
and `commands` (local commands the loop awaits — see
[Local commands](#local-commands-commands)).
