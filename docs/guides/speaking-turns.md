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
  included (tool records left out): only those the channel may know, as its
  context has them, so an event whose visibility withholds it from the channel
  never reaches the policy, nor a classifier the policy calls;
- `people` — who takes part besides the agent, by name: one name is a
  conversation between the agent and one person. These are the room's active
  participants that are neither agents nor bots, or the distinct speakers of the
  recent events when there are more (one microphone may carry several diarized
  voices);
- `speakers` — who said `event` and each of `recent`, by event id, where the room
  names them (the name the sender's transport stamped, else the participant's
  display name);
- `channel_id` — the agent's channel: `turn.by_agent(e)` tells the agent's own
  answers in `recent`.

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

## A policy on judgments

`ClassifierSpeakPolicy` asks a [classifier](classifiers.md) narrow questions
about the turn, in one call, and composes the answers in code:

```python
from roomkit import AIChannel, ClassifierSpeakPolicy, JevClassifier

policy = ClassifierSpeakPolicy(
    JevClassifier(),  # or LLMClassifier(provider), MockClassifier(...)
    agent_name="Nova",
    agent_role="the team's assistant",
    languages={"French": "Réponds en français uniquement.", "English": "Answer in English only."},
)
nova = AIChannel("nova", provider=provider, speak_policy=policy)
```

The classifier reads the agent (name, role), the people, the last `history`
turns (6) and the turn judged, each with its speaker, the agent's own under its
name. It answers these questions (`roomkit.speaking.classifier.QUESTIONS`):

| Name | Asks |
|------|------|
| `directness` | how directly the turn brings the agent in: not mentioned (0), tentatively (1), indirectly (2), directly (3) |
| `deferred` | whether the speaker postpones or declines asking |
| `unfinished` | whether the speaker stopped before saying what they want |
| `hush` | whether the turn asks the agent not to answer, or only to listen |
| `quiet_rule` | whether an earlier request to keep quiet still stands |
| `request` | whether the turn asks the agent to answer or do something now |
| `answered` | whether the turn answers a question the agent just asked |

`compose()` reads them in order:

1. **silent** when the speaker has not finished, postpones, or asks for quiet;
   silent too when a request for quiet still stands and the turn asks nothing;
2. **speak** when the turn answers the agent's question;
3. **speak** when the agent is addressed: directness 1.5 or more, or a request
   with directness 0.75 or more, or any request when one person talks with it;
4. **offer** when the speaker only wonders about it (directness 0.75 or more);
5. **silent** otherwise.

Every answer is reported in the decision's `judgments` (a choice as
`name=option`), and the decision's `reason` names the rule that decided
(`not finished`, `addressed`, `wondered about`, ...).

With `languages`, the policy also asks which language the speaker speaks, over
their recent turns: one misheard word does not switch it. The decision's notes
then carry that language's line, best written in the language itself.

The thresholds were measured with Jev's calibrated probabilities. On
`LLMClassifier` every probability is 0 or 1 and directness a whole level, which
the same rules read without change, at the cost of a generation per turn.

### Changing the questions or the rules

Replace a question by name, or add one, with `questions=`; override
`decision()` to change the composition, calling `compose()` for the rules above:

```python
from roomkit import ClassifierSpeakPolicy, ScoreQuestion, SpeakDecision


class SpeaksWhenUrgent(ClassifierSpeakPolicy):
    def decision(self, turn, answers):
        decision = super().decision(turn, answers)
        if decision.mode == "silent" and answers.score("urgency") >= 1.5:
            return SpeakDecision("speak", "urgent", decision.judgments, decision.notes)
        return decision


policy = SpeaksWhenUrgent(
    classifier,
    agent_name="Nova",
    questions={"urgency": ScoreQuestion("How urgent is `last_turn`?", ("not", "soon", "now"))},
)
```

`state()` and `questions()` are overridable too, to give the classifier more to
read. A turn without text (an image alone) is not judged: the agent speaks,
reason `nothing to judge`. A classifier that fails lets the agent speak, reason
`fallback`, as any policy. The classifier stays yours: close it when you are
done.

A runnable version is `examples/speaking_judgments.py`.

## What comes next

An agent that thinks while it listens, and offers what it wants to say when
nobody asked, builds on this policy.
