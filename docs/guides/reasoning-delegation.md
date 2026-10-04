# Reasoning Delegation (GPT-Live)

A full-duplex speech-to-speech model such as OpenAI GPT-Live holds the spoken
conversation and nothing else: it has no tools of its own. When the user asks
for something that needs a lookup, a tool, or careful thought, the model
*delegates* — it hands the work to a backend model and keeps talking while the
backend works. RoomKit calls this **reasoning delegation** (RFC §12.4.1), as
opposed to [task delegation](../features.md#agent-delegation), which dispatches
a described task to an agent in a child room.

There are two ways to run the backend.

| | Hosted backend | Integrator backend |
|---|---|---|
| Provider setting | `HostedReasoning(model=...)` | `IntegratorReasoning()` (default) |
| Who runs the model | OpenAI (Responses API) | Your `ReasoningBackend`, in your process |
| Tools | The channel's `tools`, served by `tool_handler` / `ON_TOOL_CALL` as usual | The channel's `tools`, executed by the backend through the channel gate |
| What comes back | Function results, via `submit_tool_result` | Text, relayed to the model as spoken or silent context |
| Choose it when | You want OpenAI's models and the least code | You want your own model (Claude, a local model…), your own context, or control over what the user hears |

The hosted mode needs nothing beyond the provider configuration: see
[OpenAI GPT-Live](realtime-voice-providers.md#openai-gpt-live-full-duplex).
This guide is about the integrator mode.

## How a delegation flows

```
user speaks ──► GPT-Live decides to delegate ──► session.delegation.created (no task text)
                                                          │
                       provider.on_delegation(session, delegation_id, "integrator")
                                                          │
             RealtimeVoiceChannel builds a ReasoningRequest from its transcript ledger
                                                          │
                              backend.run(request) yields ReasoningOutput(s)
                                                          │
          provider.submit_delegation_output(..., spoken=True|False) ──► commentary / thinking append
                                                          │
                          GPT-Live relays spoken output in its own words
```

The model sends no description of what it wants. The channel keeps a ledger of
the transcript (both speakers) and hands the backend everything recorded since
the previous delegation, or the whole conversation for the first one. The
backend works out the request from that.

## Configuring a backend

```python
from roomkit import AIProviderReasoningBackend, RealtimeVoiceChannel
from roomkit.providers.anthropic import AnthropicAIProvider, AnthropicConfig
from roomkit.providers.openai.live import IntegratorReasoning, OpenAILiveProvider

backend = AIProviderReasoningBackend(
    AnthropicAIProvider(AnthropicConfig(api_key="sk-ant-...", model="claude-sonnet-5")),
    system_prompt=BACKEND_INSTRUCTIONS,
    max_tool_rounds=5,          # tool rounds per delegation, the AI channel's loop bound
    spoken_progress=False,      # text before a tool round stays silent context
)

channel = RealtimeVoiceChannel(
    "voice",
    provider=OpenAILiveProvider(api_key="sk-...", delegation=IntegratorReasoning()),
    transport=transport,
    system_prompt=FRONTEND_INSTRUCTIONS,
    tools=[check_flight_status, find_alternative_flights, rebook_flight],
    tool_handler=handle_tool,
    reasoning_backend=backend,
    reasoning_timeout_s=120.0,  # a run past this is abandoned and the model told so
)
```

`AIProviderReasoningBackend` is the default backend: any `AIProvider`, made an
agent that runs on the AI channel's tool loop. It keeps its own conversation
per voice session, tool rounds included, so a second delegation sees what it
worked out for the first, and the channel releases that memory when the
session ends.

## An agent as the backend

The backend is an agent like any other, driven by the voice instead of a room.
`AgentReasoningBackend` takes one you already configured:

```python
from roomkit import Agent, AgentReasoningBackend

reasoner = Agent(
    "flight-desk",
    provider=AnthropicAIProvider(AnthropicConfig(api_key="sk-ant-...")),
    system_prompt=BACKEND_INSTRUCTIONS,
    temperature=0.2,
    max_tool_rounds=8,
    tool_loop_timeout_seconds=90.0,
)
backend = AgentReasoningBackend(reasoner, spoken_progress=False)
```

Each delegation runs on the agent's tool loop, the one every AI channel turn
runs on, with its settings (prompt, temperature, thinking, round cap, deadline,
budget) and everything that loop does at the end of a round:

- a call the provider could not parse or would not take (Gemini's
  `MALFORMED_FUNCTION_CALL`, `UNEXPECTED_TOOL_CALL`) or an empty answer after a
  tool round is asked again;
- a call whose arguments do not read is refused without running, and reported
  to the voice channel's `ON_TOOL_CALL` observers;
- the turn has its `llm.generate` span, under the voice session's span, with
  the tokens it used;
- a result larger than the agent's `evict_threshold_tokens` is stored, and the
  model reads it back with `read_stored_result`, as on any agent turn.

Its tools are the voice session's catalogue, each call served through the
voice channel's gate, which also bounds it: the voice channel's
`tool_timeout_seconds` and `tool_timeouts` apply, and a tool that waits by
design (`delegate_task`) has no bound per call, the run's `reasoning_timeout_s`
still applying. The agent's own `tool_timeout_seconds` and `tool_timeouts` do
not apply: set a backend call's bound on the voice channel. An agent that carries tools of its own (tools, skills, a
sandbox, planning, an external or human-input handler) is refused at
construction: those would run outside the gate. So is an agent registered with
a kit, whose hooks would judge each call a second time: build one for the
backend and keep it out of rooms.

A session's delegations run one at a time, each reading what the one before it
worked out. A call a delegation left unanswered (its run timed out mid-call) is
answered before the next generation, as an interrupted room turn's is.

## Spoken and silent outputs

Each `ReasoningOutput` carries a `spoken` flag:

- `spoken=True` — the model is asked to relay the text to the user, in its own
  words. It is not read verbatim.
- `spoken=False` — the text becomes silent context the model may draw on if
  the conversation turns that way.

The default backend yields the final answer spoken and the text before a tool
round silent. `spoken_progress=True` voices that intermediate text too — useful
when a job takes several slow steps and the user would otherwise wait with no
news. Outputs longer than the API's per-append bound are split on sentence
boundaries by the provider.

## Two prompts

The live model and the backend read different instructions. The live model's
prompt (`system_prompt` on the channel) says how to converse and *when* to
delegate: "Answer simple questions directly. Delegate anything about the
user's flights. Relay what the backend sends as it arrives; ignore results the
conversation has moved past." The backend's prompt (`system_prompt` on the
backend) says *what to do* with a transcript: "Each message is the recent
voice conversation. Work out what is asked. Use the tools. Reply in plain
conversational text, never claim an action completed without a tool result."

## Tool calls go through the channel gate

A backend's tool calls are tool calls of the framework. `ReasoningRequest`
carries the channel's declared catalogue (`tools`), the session's tools the
backend's model is not offered with the refusal each reads (`unavailable`: the
tool policy's, or a skill's worded for a model that cannot activate one), both
read when the channel hands the delegation over, with the participant's role
as it stands then (a participant promoted mid-session is offered the tool by
the delegations handed over after it; one the backend queues keeps what was
read, and the gate still judges its calls), and two callables that run one
call through the same steps as any realtime tool
call — the pre-execution gate (declared catalogue, tool policy, skill gating,
argument schema, `BEFORE_TOOL_USE`),
the channel's `tool_handler` inside the tool call context, `ON_TOOL_CALL` and
the bound on the result:

- `execute_tool_call(name, arguments)` returns a `ToolCallResult`: the `text`
  the model reads and `is_error`, set when the call was refused, failed, was
  blocked, was served by nothing or was cancelled, and `refused`, set when
  among those the call was refused. Prefer it: the model then reads a failed
  call as one, as every tool loop marks it, and an agent backend's loop reads
  a refusal as refused and any other error as failed.
- `execute_tool(name, arguments)` returns the text alone.

A `BEFORE_TOOL_USE` hook that blocks `rebook_flight` blocks it for the backend
too, and the block's reason is what the backend's model reads. A delegation is
not a way around the gate. The built-in `AIProviderReasoningBackend` uses
`execute_tool_call`.

## When the backend cannot answer

A full-duplex model keeps the conversation open until its delegation says
something. The channel therefore always answers, with one spoken output:

| Situation | What the model is told |
|---|---|
| No `reasoning_backend` configured | "No backend is available to handle delegated work in this session." |
| The backend yielded nothing (or only blank text) | "The delegated work finished without an answer." |
| The backend raised | "The delegated work could not be completed." |
| The backend's turn did not complete (round cap, deadline, budget, an answer cut or empty) | "The delegated work could not be completed." |
| The run exceeded `reasoning_timeout_s` | "The delegated work took too long and was abandoned." |

A turn that did not complete has no answer: the built-in backends yield what
the model said before each tool round as progress, never as the answer, and
raise `ReasoningCutShortError` (its `reason` is the loop's
`loop_end_reason`), which the channel answers as a failed backend. A backend
call the run's timeout cut reaches `ON_TOOL_CALL` once, `cancelled`.

A running delegation counts as activity for `wait_idle()`, and ending the
session cancels it.

## Writing your own backend

```python
from collections.abc import AsyncIterator

from roomkit import ReasoningBackend, ReasoningOutput, ReasoningRequest


class FlightDeskBackend(ReasoningBackend):
    async def run(self, request: ReasoningRequest) -> AsyncIterator[ReasoningOutput]:
        last_user = next((l.text for l in reversed(request.transcript) if l.role == "user"), "")
        yield ReasoningOutput("Let me look that up.", spoken=False)
        status = await request.execute_tool_call(
            "check_flight_status", {"flight_number": parse(last_user)}
        )
        if status.is_error:
            yield ReasoningOutput("I could not check that flight.", spoken=True, is_final=True)
            return
        yield ReasoningOutput(f"Here is what I found: {status.text}", spoken=True, is_final=True)

    async def session_ended(self, session_id: str) -> None:
        ...  # drop any per-session state
```

`run()` is an async generator: yield outputs as you learn things. Route every
tool call through `request.execute_tool_call` (or `request.execute_tool`). A
call your own loop refuses before the gate (its arguments did not parse) goes
to `request.report_refusal(name, arguments, body)`, so the channel's
`ON_TOOL_CALL` observers see it as they see every refused call; pass
`cancelled=True` for a call your loop cut, or `refused=False` (and
`detail=`, what failed) for one that failed. A call your provider ran
itself (a provider-side tool such as web search) goes to
`request.report_call(name, arguments, result, is_error=..., detail=...,
tool_call_id=...)`: the channel's `ON_TOOL_CALL` hooks hear it once, as they
hear a provider-side call on an AI channel. `AgentReasoningBackend` does both
for you. Implement `session_ended()` if you keep state per session and
`close()` if you hold resources.

## Observability

`ON_REALTIME_DELEGATION` fires for every delegation, hosted or integrator, with
a `RealtimeDelegationEvent` (`session`, `delegation_id`, `target`). The time
from it to the first spoken output is the latency the user hears.

```python
@kit.hook(HookTrigger.ON_REALTIME_DELEGATION, execution=HookExecution.ASYNC)
async def on_delegation(event, ctx):
    logger.info("delegation %s → %s", event.delegation_id, event.target)
```

Backend tool calls appear in `ON_TOOL_CALL` like any other, with a
`tool_call_id` prefixed by the delegation id.

## Example

`examples/realtime_voice_local_openai_live_backend.py` runs GPT-Live with a
Claude backend and three slow rebooking tools, from the local microphone.
