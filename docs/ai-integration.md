# AI Assistant Integration

RoomKit publishes its documentation in forms a coding assistant (Claude Code,
Cursor, Copilot…) can read, so the code it writes uses RoomKit's actual API
rather than a guess from its training data.

## The files

| File | What it holds | Where |
|---|---|---|
| `llms.txt` | Index of the documentation, one line per page, in the [llms.txt format](https://llmstxt.org/) | [www.roomkit.live/docs/llms.txt](https://www.roomkit.live/docs/llms.txt), and inside the installed package |
| `llms-full.txt` | The topic pages (architecture, channels, hooks, voice, orchestration, API) in one file | [www.roomkit.live/docs/llms-full.txt](https://www.roomkit.live/docs/llms-full.txt), and inside the installed package |
| `AGENTS.md` | Conventions for working on RoomKit itself: commands, project layout, patterns, code style, what to ask first | [repository root](https://github.com/roomkit-live/roomkit/blob/main/AGENTS.md), and inside the installed package |

Give the assistant `llms.txt` or `llms-full.txt` to build an application with
RoomKit; `AGENTS.md` is for contributing to RoomKit.

## From Python

The package ships the copies that match the installed version:

```python
from roomkit import get_agents_md, get_ai_context, get_llms_full_txt, get_llms_txt

index = get_llms_txt()          # llms.txt
full = get_llms_full_txt()      # llms-full.txt
guidelines = get_agents_md()    # AGENTS.md
context = get_ai_context()      # AGENTS.md and llms.txt in one string
```

## Agent Skills

[roomkit-skills](https://github.com/roomkit-live/roomkit-skills) holds skills
that walk an assistant through common tasks: setting up a project, creating a
room, an AI channel, a voice agent, an SMS or WhatsApp channel, hooks,
orchestration, PostgreSQL and telemetry. For Claude Code:

```bash
claude plugin marketplace add https://github.com/roomkit-live/roomkit-skills
claude plugin install roomkit-dev
```

## MCP

RoomKit's AI channels can call tools from MCP servers; see
[MCP Integration](mcp.md).
