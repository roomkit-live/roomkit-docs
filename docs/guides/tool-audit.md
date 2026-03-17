# Tool Auditing

Record every tool call with input, output, timing, and status. Built-in implementations write to JSONL files or log to the console. Plugs into `AIChannel` and `RealtimeVoiceChannel` automatically.

## Quick start

```python
from roomkit import Agent, JSONLToolAuditor, audit_tool_handler

# Create an auditor
auditor = JSONLToolAuditor("/tmp/audit.jsonl")

# Wrap any tool handler — recording is automatic
async def my_handler(name, args):
    return '{"status": "ok"}'

audited = audit_tool_handler(my_handler, auditor, agent_id="my-agent")

# Use with an Agent
agent = Agent(
    "my-agent",
    provider=my_provider,
    tool_handler=audited,
    tools=[...],
)

# After the session, print summary
auditor.print_summary()
```

## What gets recorded

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

## Implementations

### JSONLToolAuditor

Writes entries to a JSONL file and keeps them in memory:

```python
from roomkit import JSONLToolAuditor

auditor = JSONLToolAuditor("/tmp/screen_ai/audit.jsonl")
```

### ConsoleToolAuditor

Logs entries in real-time via Python logging:

```python
from roomkit import ConsoleToolAuditor

auditor = ConsoleToolAuditor()
# Logs: [AUDIT] [+] exec.search_google → ok (9333ms) {"status": "ok", ...}
```

### Custom auditor

Implement the `ToolAuditor` ABC:

```python
from roomkit import ToolAuditor, ToolAuditEntry

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

## Two handler wrappers

### For AIChannel (standard tool handler)

```python
from roomkit import audit_tool_handler

# Wraps async (name, args) -> str
audited = audit_tool_handler(handler, auditor, agent_id="exec")
agent = Agent("exec", tool_handler=audited, ...)
```

### For RealtimeVoiceChannel (voice tool handler)

```python
from roomkit.orchestration.tool_audit import audit_realtime_tool_handler

# Wraps async (session, name, args) -> dict|str
audited = audit_realtime_tool_handler(handler, auditor, agent_id="voice")
voice_channel = RealtimeVoiceChannel("voice", tool_handler=audited, ...)
```

## Print summary

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
