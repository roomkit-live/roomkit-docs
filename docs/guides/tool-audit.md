# Auditing

RoomKit provides two levels of auditing: **tool auditing** for recording individual tool calls, and **session auditing** for capturing the full conversation timeline including speech turns, tool calls, vision events, and interruptions.

## Session Auditing

`JSONLSessionAuditor` captures the complete conversation flow — not just tools.

### Quick start

```python
from roomkit import RoomKit
from roomkit.orchestration.session_audit import JSONLSessionAuditor

auditor = JSONLSessionAuditor("/tmp/session.jsonl")
kit = RoomKit()

# Auto-capture speech, vision, barge-in, session events
auditor.attach(kit)

# ... set up channels, rooms, sessions ...

# Tool calls are recorded manually from your handler:
from roomkit.orchestration.tool_audit import ToolAuditEntry

auditor.record_tool(ToolAuditEntry(
    ts=datetime.now().isoformat(),
    agent_id="my-agent",
    tool_name="search",
    arguments={"q": "roomkit"},
    result="Found 3 results",
    status="ok",
    duration_ms=1500,
))

# After the session
auditor.print_summary()
```

### What gets captured

| Event type | Hook trigger | What it records |
|---|---|---|
| `speech` | `ON_TRANSCRIPTION` | User and assistant voice turns (final only) |
| `tool_call` | Manual `record_tool()` | Tool name, args, result, duration, status |
| `vision` | `ON_VISION_RESULT` | Periodic screen/camera descriptions |
| `barge_in` | `ON_BARGE_IN` | User interruptions |
| `session` | `ON_SESSION_STARTED` | Voice session lifecycle |

### Transcript output

```
Session Audit (/tmp/session.jsonl)
============================================================
  [11:56:02] SESSION Session started
  [11:56:05] USER: "Open Google Chrome and search for roomkit"
  [11:56:07] ASSISTANT: "Sure, let me check your screen first."
  [11:56:08] TOOL describe_screen → OK (5886ms)
           Chrome with Google search page open
  [11:56:14] VISION Chrome browser showing search results
  [11:56:30] BARGE-IN User interrupted
  [11:57:42] ASSISTANT: "Done! The roomkit.live website is open."

  Duration: 1m 40s | Turns: 2 user, 2 assistant
  Tool calls: 1 (5886ms) | Vision: 1 | Interruptions: 1
  Tools: describe_screen(1)
```

### ToolAuditor compatibility

`SessionAuditor` works as a drop-in for APIs that expect a `ToolAuditor`:

```python
# Use the bridge property
bridge = auditor.tool_auditor
audited = audit_tool_handler(handler, bridge, agent_id="my-agent")
```

### Custom session auditor

Implement the `SessionAuditor` ABC:

```python
from roomkit.orchestration.session_audit import SessionAuditor, SessionAuditEntry

class MySessionAuditor(SessionAuditor):
    def record(self, entry: SessionAuditEntry) -> None:
        send_to_monitoring(entry.model_dump())

    @property
    def entries(self) -> list[SessionAuditEntry]:
        return []

    def summary(self) -> str:
        return "See monitoring dashboard"
```

---

## Tool Auditing

For simpler use cases that only need tool call recording, use `JSONLToolAuditor` directly.

### Quick start

```python
from roomkit import Agent, Tool
from roomkit.orchestration.tool_audit import JSONLToolAuditor, audit_tool_handler

# Create an auditor
auditor = JSONLToolAuditor("/tmp/audit.jsonl")

# Wrap any tool handler — recording is automatic
async def my_handler(name, args):
    return '{"status": "ok"}'

audited = audit_tool_handler(my_handler, auditor, agent_id="my-agent")

# Use with an Agent — pass Tool objects via tools=[], audited handler via tool_handler=
agent = Agent(
    "my-agent",
    provider=my_provider,
    tools=[my_tool],
    tool_handler=audited,
)

# After the session, print summary
auditor.print_summary()
```

### What gets recorded

Every tool call produces a `ToolAuditEntry`:

| Field | Type | Description |
|-------|------|-------------|
| `ts` | str | ISO timestamp |
| `agent_id` | str | Which agent made the call |
| `tool_name` | str | Tool function name |
| `arguments` | dict | Input arguments |
| `result` | str | Tool output (truncated to 500 chars) |
| `status` | str | `ok`, `failed`, or `error` |
| `duration_ms` | float | Execution time in milliseconds |
| `metadata` | dict | Optional extra data |

Status is auto-detected: if the tool returns `{"status": "failed"}`, the entry is marked as failed.

### Implementations

#### JSONLToolAuditor

Writes entries to a JSONL file and keeps them in memory:

```python
from roomkit.orchestration.tool_audit import JSONLToolAuditor

auditor = JSONLToolAuditor("/tmp/screen_ai/audit.jsonl")
```

#### ConsoleToolAuditor

Logs entries in real-time via Python logging:

```python
from roomkit.orchestration.tool_audit import ConsoleToolAuditor

auditor = ConsoleToolAuditor()
# Logs: [AUDIT] [+] exec.search_google → ok (9333ms) {"status": "ok", ...}
```

#### Custom auditor

Implement the `ToolAuditor` ABC:

```python
from roomkit.orchestration.tool_audit import ToolAuditor, ToolAuditEntry

class MyAuditor(ToolAuditor):
    def record(self, entry: ToolAuditEntry) -> None:
        # Send to your monitoring system
        send_to_datadog(entry.model_dump())

    @property
    def entries(self) -> list[ToolAuditEntry]:
        return []

    def summary(self) -> str:
        return "See Datadog dashboard"
```

### Handler wrappers

#### For AIChannel (standard tool handler)

```python
from roomkit.orchestration.tool_audit import audit_tool_handler

# Wraps async (name, args) -> str
audited = audit_tool_handler(handler, auditor, agent_id="exec")
agent = Agent("exec", tools=[my_tool], tool_handler=audited, ...)
```

#### For RealtimeVoiceChannel (voice tool handler)

The same `audit_tool_handler` works for voice — the tool handler signature is unified as `async (name: str, args: dict) -> str` across all channels.

```python
audited = audit_tool_handler(handler, auditor, agent_id="voice")
voice_channel = RealtimeVoiceChannel("voice", tools=[my_tool], tool_handler=audited, ...)
```

### Print summary

```python
auditor.print_summary()
```

```
Tool Audit (/tmp/screen_ai/audit.jsonl)
============================================================
   1. [OK] search_google  (9333ms)
      → {"status": "ok", "query": "roomkit conversation AI", "results": [...]}
   2. [OK] click_result  (5505ms)
      → {"status": "ok", "clicked": "roomkit.live"}
   3. [OK] read_screen  (2729ms)
      → {"status": "ok", "description": "RoomKit homepage..."}

  Total: 3 calls, 17567ms
  Tools: click_result(1), read_screen(1), search_google(1)
```
