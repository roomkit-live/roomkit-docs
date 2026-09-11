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
    max_tool_rounds=5,          # generate → tools → generate, at most this many rounds
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

`AIProviderReasoningBackend` is the default backend: any `AIProvider` run
through a small tool loop. It keeps its own conversation per voice session, so
a second delegation sees what it worked out for the first, and the channel
releases that memory when the session ends.

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
carries the channel's declared catalogue (`tools`) and an `execute_tool`
callable that runs one call through the same pre-execution gate as a realtime
tool call — declared catalogue, argument schema, skill gating,
`BEFORE_TOOL_USE` — then the channel's `tool_handler`, `ON_TOOL_CALL` and
result truncation. A `BEFORE_TOOL_USE` hook that blocks `rebook_flight` blocks
it for the backend too, and the denial text is what the backend's model reads.
A delegation is not a way around the gate.

## When the backend cannot answer

A full-duplex model keeps the conversation open until its delegation says
something. The channel therefore always answers, with one spoken output:

| Situation | What the model is told |
|---|---|
| No `reasoning_backend` configured | "No backend is available to handle delegated work in this session." |
| The backend yielded nothing (or only blank text) | "The delegated work finished without an answer." |
| The backend raised | "The delegated work could not be completed." |
| The run exceeded `reasoning_timeout_s` | "The delegated work took too long and was abandoned." |

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
        status = await request.execute_tool("check_flight_status", {"flight_number": parse(last_user)})
        yield ReasoningOutput(f"Here is what I found: {status}", spoken=True, is_final=True)

    async def session_ended(self, session_id: str) -> None:
        ...  # drop any per-session state
```

`run()` is an async generator: yield outputs as you learn things. Route every
tool call through `request.execute_tool`. Implement `session_ended()` if you
keep state per session and `close()` if you hold resources.

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
