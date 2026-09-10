# Agent Delegation

Delegate tasks to background agents while conversations continue. A voice agent can hand off a PR review to a specialist while still chatting with the user — one call replaces ~100 lines of manual boilerplate.

## Quick start

```python
from roomkit import RoomKit, ChannelCategory
from roomkit.channels.agent import Agent
from roomkit.channels.ai import AIChannel

kit = RoomKit()

# Register a front-facing agent and a background specialist
voice_agent = AIChannel("voice-assistant", provider=my_ai, system_prompt="...")
pr_reviewer = Agent("pr-reviewer", provider=reviewer_ai, role="PR Reviewer")

kit.register_channel(voice_agent)
kit.register_channel(pr_reviewer)

# Set up a room
await kit.create_room(room_id="call-room")
await kit.attach_channel("call-room", "voice-assistant", category=ChannelCategory.INTELLIGENCE)

# Delegate — one call does everything
task = await kit.delegate(
    room_id="call-room",
    agent_id="pr-reviewer",
    task="Review the latest PR on roomkit",
    notify="voice-assistant",
)

# Fire and forget, or block for the result
result = await task.wait(timeout=30.0)
print(result.output)
```

## How it works

`kit.delegate()` creates a **child room** linked to the parent:

```
Parent room (call-room)
  ├── voice-call        (transport)
  ├── voice-assistant   (intelligence)  ← notified when done
  └── email-out         (transport)     ← shared to child

Child room (call-room::task-a1b2c3d4)
  ├── pr-reviewer       (intelligence)  ← runs the task
  └── email-out         (transport)     ← shared from parent
```

The flow:

1. **Child room created** with parent link in metadata
2. **Agent attached** as `INTELLIGENCE` in the child room
3. **Channels shared** from parent (same provider instance, different binding)
4. **Task event injected** into child room → agent picks it up
5. **Agent response collected** as the task result
6. **Parent notified** via system prompt injection on the `notify` channel
7. **Hooks fired**: `ON_TASK_DELEGATED` (immediately) and `ON_TASK_COMPLETED` (on finish)

## Fire and forget

```python
task = await kit.delegate(
    room_id="call-room",
    agent_id="pr-reviewer",
    task="Review the latest PR",
)
# Returns immediately — task runs in the background
```

## Blocking for result

```python
task = await kit.delegate(room_id="call-room", agent_id="pr-reviewer", task="...")
result = await task.wait(timeout=30.0)

if result.status == "completed":
    print(result.output)
else:
    print(f"Failed: {result.error}")
```

## Parallel delegation

```python
import asyncio

task_a = await kit.delegate(room_id="room-1", agent_id="reviewer", task="Review PR")
task_b = await kit.delegate(room_id="room-1", agent_id="analyst", task="Analyze metrics")

result_a, result_b = await asyncio.gather(
    task_a.wait(timeout=30.0),
    task_b.wait(timeout=30.0),
)
```

## Tool integration

Let the AI decide when to delegate using `setup_delegation()`:

```python
from roomkit.tasks import DelegateHandler, setup_delegation, build_delegate_tool

handler = DelegateHandler(kit, notify="voice-assistant")

# Constrain which agents the AI can delegate to
tool = build_delegate_tool([
    ("pr-reviewer", "Reviews GitHub PRs"),
    ("code-writer", "Writes code from specs"),
])

setup_delegation(voice_agent, handler, tool=tool)
```

The AI will see a `delegate_task` tool and can call it naturally:

```json
{
  "name": "delegate_task",
  "arguments": {
    "agent": "pr-reviewer",
    "task": "Review PR #42 and summarize findings",
    "share_channels": ["email-out"]
  }
}
```

## Delegation for RealtimeVoiceChannel

For realtime voice agents (Gemini Live, OpenAI Realtime), use `setup_realtime_delegation()` instead. It resolves the room ID from the current voice session context:

```python
from roomkit.tasks import DelegateHandler, setup_realtime_delegation, build_delegate_tool
from roomkit.channels.realtime_voice import RealtimeVoiceChannel

voice = RealtimeVoiceChannel("voice", provider=realtime_provider, transport=backend)

handler = DelegateHandler(kit, notify="voice")
tool = build_delegate_tool([("exec-agent", "Runs tasks on screen")])

setup_realtime_delegation(voice, handler, tool=tool)
```

Under the hood, this injects the delegate tool dict into `channel._tools` and wraps `_tool_handler` to intercept `delegate_task` calls. Room ID is resolved via `get_current_voice_session()` + `channel.session_rooms`.

## Preventing re-delegation (dedup)

When a task completes and the result is delivered back, the AI may try to delegate the same task again. Use `CompletedTaskCache` to prevent this:

```python
from roomkit.tasks import DelegateHandler, CompletedTaskCache

cache = CompletedTaskCache(ttl_seconds=300)  # 5 min TTL

handler = DelegateHandler(
    kit,
    cache=cache,              # dedup: return cached result instead of re-delegating
    serialize_per_room=True,  # only one delegation at a time per room
)
```

This enables three features:

- **Gap 13 — Dedup**: If a matching `(room_id, agent_id, task_hash)` was recently completed, the cached result is returned with `"from_cache": True` instead of spawning a new agent.
- **Gap 14 — Serialization**: With `serialize_per_room=True`, concurrent delegation attempts for the same room are queued. Only one runs at a time, preventing screen/resource conflicts.
- **Gap 15 — Context injection**: Recent task descriptions are automatically injected into the new delegation's `context["previous_tasks"]` so the background agent knows what was already done.

## Shared channels

Channels shared from the parent use the **same provider instance** with a different binding:

```python
task = await kit.delegate(
    room_id="call-room",
    agent_id="pr-reviewer",
    task="Review PR and email summary",
    share_channels=["email-out"],  # same EmailChannel, shared to child
)
```

The background agent can send emails through the shared channel just like the parent.

## Hooks

Two hook triggers for observability:

```python
@kit.hook(HookTrigger.ON_TASK_DELEGATED, execution=HookExecution.ASYNC)
async def on_delegated(event, ctx):
    task_id = event.metadata["task_id"]
    agent_id = event.metadata["agent_id"]
    logger.info("Task %s delegated to %s", task_id, agent_id)

@kit.hook(HookTrigger.ON_TASK_COMPLETED, execution=HookExecution.ASYNC)
async def on_completed(event, ctx):
    task_id = event.metadata["task_id"]
    status = event.metadata["task_status"]
    duration = event.metadata["duration_ms"]
    logger.info("Task %s: %s in %.0fms", task_id, status, duration)
```

## Callbacks

For programmatic handling beyond hooks:

```python
async def handle_result(result):
    if result.status == "completed":
        await send_notification(result.output)

task = await kit.delegate(
    room_id="call-room",
    agent_id="pr-reviewer",
    task="Review PR",
    on_complete=handle_result,
)
```

## Attributing a direct inbound response

Hosts that dispatch work through `process_inbound()` can read
`InboundResult.response_events` to identify the responses belonging to that call.
Reading the room after the inbound index can include another turn that committed
or completed while the host was waiting.

```python
result = await kit.process_inbound(message, room_id="worker-room", defer_delivery=True)
if result.delivery is not None:
    result = await result.delivery.wait()

if not result.blocked and result.error is None:
    replies = [
        event for event in result.response_events
        if isinstance(event.content, TextContent)
        and event.source.channel_id == "worker"
    ]
    if replies:
        answer = max(replies, key=lambda event: event.index).content.body
```

Import `TextContent` from `roomkit`. The collection contains persisted, delivered
response events from the call's delivery cascade, including streamed segments.
It excludes the initial inbound, blocked responses and other calls' events;
broadcast-hook edits are reflected in the returned content. Events retain their
indices, source and visibility. A host still applies its own access policy when
presenting them to a user. An empty collection means no persisted response, and
a partial response does not override `result.error` or the turn's end metadata.

With deferred delivery, wait for `result.delivery.wait()` before treating the
collection as complete. The runnable
[`deferred_inbound.py` example](https://github.com/roomkit-live/roomkit/blob/main/examples/deferred_inbound.py)
reads the first call's answer after a second call has completed, without credentials.

For a voice destination that is reused across calls, pass the original call's
unique `channel_id` to `kit.deliver()`. A room-only destination auto-selects its
current transport; that may be a different call by the time background work ends.
The host owns the call lifetime and may keep the result in its text conversation
when that voice channel has ended.

## Delivery strategies

Control how task results are delivered back to the parent conversation:

```python
from roomkit.tasks import WaitForIdleDelivery, ImmediateDelivery

# Wait for TTS playback to finish, then deliver
kit = RoomKit(delivery_strategy=WaitForIdleDelivery())

# Deliver immediately (may interrupt)
kit = RoomKit(delivery_strategy=ImmediateDelivery(prompt="Task done!"))
```

| Strategy | Behavior |
|----------|----------|
| `ContextOnlyDelivery` | Inject into system prompt, wait for next user turn (default) |
| `ImmediateDelivery` | Send synthetic inbound message immediately |
| `WaitForIdleDelivery` | Wait for TTS playback to finish, then send |

All strategies support `RealtimeVoiceChannel` — they detect the channel type and deliver via `inject_text()` instead of `process_inbound()`. For `WaitForIdleDelivery`, realtime voice delivery is immediate since there's no playback queue to wait for.

## Structured results

By default a delegated agent's result is its **free-text response**, scraped
from the child room. Set `require_structured_result=True` to instead force the
agent to hand its work back by calling a `submit_result` tool — a structured,
parseable handoff, and a guarantee of a result at all (the worker can't punt
with a question back to the user):

```python
task = await kit.delegate(
    room_id="call-room",
    agent_id="pr-reviewer",
    task="Review the latest PR on roomkit",
    require_structured_result=True,
    wait=True,
)
# result.output is the JSON-encoded submit_result payload
```

The framework injects the `submit_result` tool, runs the agent, and applies a
completion guard: if the agent ends a turn without calling it, the agent is
re-prompted to use the tool (up to `max_result_retries` times); if it still
hasn't, the orchestration submits a failure on its behalf
(`status="failed"`, `by="orchestration"`) carrying the worker's last raw output
so the caller knows what went wrong.

The `submit_result` payload:

| Field | Type | Description |
|-------|------|-------------|
| `status` | `"completed"` / `"failed"` | Required. `failed` only if the worker genuinely could not do the task |
| `summary` | `str` | Required. One or two sentences on what was produced |
| `data` | `object` | Structured result for the next step to build on |
| `deliverables` | `list[{title, url}]` | Concrete artifacts (e.g. a published report URL) |
| `reason` | `str` | If `failed`, why |

Capture is delivery-agnostic: a function-calling provider's tool call is caught
directly, and a `claude_code` worker (which calls the gateway-exposed tool,
surfaced with an `mcp__…` prefix) is recovered by scanning its persisted trace.
The supervised sequential flow (see
[Multi-Agent Orchestration](orchestration.md#supervised-validation-sequential))
uses this internally so every worker hands the supervisor a clean object to
review.

## Inherited context

A child room inherits the parent room's delegation context envelope: any
context attached to the parent cascades verbatim onto each child room (and
re-stamps so it carries into nested delegations). A worker delegated from
within another delegated task therefore sees the same originating context as
the top-level parent.

## Configuration reference

| Parameter | Type | Description |
|-----------|------|-------------|
| `room_id` | `str` | Parent room ID (required) |
| `agent_id` | `str` | Channel ID of the background agent (required) |
| `task` | `str` | What the agent should do (required) |
| `context` | `dict` | Optional context passed to the agent |
| `share_channels` | `list[str]` | Channel IDs to share from parent |
| `notify` | `str` | Channel ID to update with result (default: `agent_id`) |
| `on_complete` | `callable` | Async callback `(DelegatedTaskResult) -> None` |
| `wait` | `bool` | Run inline and return a pre-completed task (default `False` = background) |
| `require_structured_result` | `bool` | Force the agent to hand back via the `submit_result` tool (default `False`) — see [Structured results](#structured-results) |
| `max_result_retries` | `int` | Re-prompts for `submit_result` before the orchestration fails on the agent's behalf (default `3`) |

## Custom task runner

The default `InMemoryTaskRunner` uses `asyncio.create_task()`. For distributed deployments, implement the `TaskRunner` ABC:

```python
from roomkit.tasks import TaskRunner, DelegatedTask

class RedisTaskRunner(TaskRunner):
    async def submit(self, kit, task, *, context=None, on_complete=None):
        # Submit to Redis queue
        ...

    async def cancel(self, task_id):
        # Cancel via Redis
        ...

    async def close(self):
        # Shutdown
        ...

kit = RoomKit(task_runner=RedisTaskRunner(redis_url="..."))
```


When a background result follows a realtime tool's spoken acknowledgement, use
`strategy="wait_for_idle"` with that explicit channel ID. Realtime idle detection
includes pending tool calls and the provider response after submission, then
waits for its audio to reach the transport. `WaitForIdle` retains its bounded
playback timeout and best-effort fallback; it is not a guarantee that a model
will produce an acknowledgement or a result. Inspect the voice trace and audio
when verifying that conversational contract.
