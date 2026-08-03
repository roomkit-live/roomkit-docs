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

During a streamed response, thinking (when `show_thinking=True`) renders dim
above the answer, and tool calls render inline:

```text
💭 The user wants a summary — checking the docs first.
⏺ search({"query": "roomkit"})
  ⎿ ✓ search · 42 ms
● Assistant
# Summary
...
```

### The pinned input bar

On a real terminal, `run()` starts an interactive shell instead of the plain
read-print loop. The input zone is pinned at the bottom of the screen —
framed by two full-width rules, with a status toolbar under it — and the
conversation scrolls above in normal scrollback. Submitted lines are echoed
into the transcript with a full-width tinted background and a muted chevron,
so your messages stand apart from agent output:

```text
❯ summarize the last meeting          ← echoed, tinted background
● Assistant
Here is the summary you asked for — …

──────────────────────────────────────────────
❯ and now compare it with│
──────────────────────────────────────────────
 demo · claude-sonnet-5 (anthropic) · working (1 queued)
```

- **Type while the agent streams.** The bar never blocks. Each submitted
  line queues, and queued messages process strictly one at a time — the
  toolbar shows `working (n queued)` until the queue drains.
- **Rendering granularity.** Under the bar the response flushes per
  completed Markdown block (paragraph, list, fenced code block — fences are
  never split mid-block). In classic and non-TTY modes streaming stays
  token-level.
- **Exiting.** `quit` / `exit` / `q`, Ctrl+D, or Ctrl+C end the session. An
  in-flight turn is cancelled and pending queued messages are dropped; a
  turn cancelled mid-stream does not persist its partial text.

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
[Identity Resolution](identity-resolution.md)), `welcome`, and
`content_factory` (map a raw input line to custom content, or return `None`
to swallow a line — the hook point for local slash commands).
