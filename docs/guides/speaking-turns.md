# Speaking Turns

An agent in a conversation with several people, or listening to one person who
thinks aloud, should not answer every turn. Give its `AIChannel` a **speak
policy** and the channel asks it, once per event it would answer, whether the
agent speaks now, offers to, or stays silent (RFC §6.4).

```python
from roomkit import AIChannel, SpeakDecision, SpeakPolicy, SpeakTurn


class SpeaksWhenNamed(SpeakPolicy):
    async def decide(self, turn: SpeakTurn) -> SpeakDecision:
        text = turn.event.content.body.lower()
        if "nova" in text:
            return SpeakDecision("speak", reason="named", judgments={"named": 1.0})
        return SpeakDecision("silent", reason="not for me", judgments={"named": 0.0})


nova = AIChannel("nova", provider=provider, speak_policy=SpeaksWhenNamed())
```

A runnable version, with two people and the three modes, is
`examples/speaking_turns.py`.

## The three modes

| Mode | What the channel does |
|------|-----------------------|
| `speak` | Runs the turn as without a policy. The decision's `notes` join the turn's notes before `BEFORE_AI_GENERATION`, which sees them. |
| `offer` | Runs the turn with one more note: offer, in one short sentence, what the agent could add, without giving it. |
| `silent` | Runs no turn: no generation, `BEFORE_AI_GENERATION` does not fire, nothing is delivered. The event is stored, and handed to the memory provider, as any event: the conversation is the agent's memory whether it spoke or not. |

## What a policy reads

`SpeakTurn` carries:

- `event` — the event the turn would answer;
- `recent` — the room's messages before it, oldest first, the agent's own answers
  included (tool records left out);
- `people` — who takes part besides the agent, by display name (or id): one name
  is a conversation between the agent and one person.

The agent's own identity is the policy's: a policy that judges whether the agent
was addressed takes its name in its constructor.

## What a policy returns

`SpeakDecision(mode, reason="", judgments={}, notes=())`:

- `reason` — why, in a few words, for logs and the hook;
- `judgments` — what the policy weighed, by name: what makes a decision
  measurable, turn by turn;
- `notes` — blocks for the turn's notes when the agent speaks or offers, such as
  "Answer in French" or what the agent was thinking.

## What is never submitted to it

- **Instructions**, a delegated task's hand-back among them: the application
  asked for that turn.
- **A turn a room's strategy takes** (a loop, a supervisor): the strategy decides.
- **The channel's own events** and tool records.

## A policy that fails does not silence the agent

The channel bounds the wait for a decision, `speak_timeout=2.0` seconds by
default. A policy that raises, or does not decide in time, is logged, and the
agent speaks: the decision reported is `speak` with the reason `fallback`.

## Following the decisions

Every decision fires `ON_SPEAK_DECISION` (async), with the room, the channel, the
event and the decision:

```python
from roomkit import HookExecution, HookTrigger, SpeakDecisionEvent


@kit.hook(HookTrigger.ON_SPEAK_DECISION, execution=HookExecution.ASYNC)
async def on_decision(event: SpeakDecisionEvent, ctx) -> None:
    logger.info("%s %s %s", event.decision.mode, event.decision.reason, event.decision.judgments)
```

A channel without a policy answers every event, as before, and decides nothing.
`AlwaysSpeak` answers every event and reports each decision: a baseline to
measure another policy against.

## Testing

`MockSpeakPolicy` returns scripted decisions in order (the last one repeats),
after an optional delay, or raises a given error, and records the turns it was
asked about:

```python
from roomkit import MockSpeakPolicy, SpeakDecision

policy = MockSpeakPolicy(["silent", SpeakDecision("speak", notes=("Answer briefly.",))])
```

## What comes next

The policy above reads keywords. Policies built on narrow judgments (was the
agent addressed, did the person finish, did they ask the agent to stay quiet),
answered by a classifier and composed in code, and an agent that thinks while
it listens, build on this seam.
