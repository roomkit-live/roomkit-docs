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
event and the decision, how long the policy took to decide (`duration_ms`: the
channel's bound when it did not decide in time) and whether the policy was asked
again once the agent thought (`asked_again`, see [thinking while
listening](#thinking-while-listening)). A policy is measured from the hook, without
wrapping it:

```python
from roomkit import HookExecution, HookTrigger, SpeakDecisionEvent


@kit.hook(HookTrigger.ON_SPEAK_DECISION, execution=HookExecution.ASYNC)
async def on_decision(event: SpeakDecisionEvent, ctx) -> None:
    again = " (asked again)" if event.asked_again else ""
    logger.info(
        "%s %s in %d ms%s %s",
        event.decision.mode,
        event.decision.reason,
        event.duration_ms,
        again,
        event.decision.judgments,
    )
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
    languages={"English": "Answer in English only."},
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
then carry that language's line. Give one entry per language the agent answers
in, each line written in its own language: a note in English around a request
in another language pulls the model into English.

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

## Answering only some people

An agent may listen to everyone in a room and answer only some of them: a
television on in the living room, a meeting where it assists one person.
`AnswerOnly` wraps any policy:

```python
from roomkit import AnswerOnly, ClassifierSpeakPolicy

nova = AIChannel(
    "nova",
    provider=provider,
    speak_policy=AnswerOnly(
        ClassifierSpeakPolicy(classifier, agent_name="Nova"), people=["Sylvain"]
    ),
)
```

- A turn from anyone else, or from a speaker the room does not name, is
  `silent` with the reason `only listened to`, and the policy it wraps is not
  asked (no classifier call). The turn is stored, and a thinker thinks about it:
  the agent hears it. Asked again once it thought, it stays silent.
- A turn from one of the people is the wrapped policy's, with only those people
  in `SpeakTurn.people`, the speaker among them even when the room's
  participant record names the microphone otherwise: Sylvain alone in front of
  the television is a conversation with one person, not a group.
- A speaker is matched by the name the room gives them (the name the sender's
  transport stamped, else the participant's display name), ignoring case and
  spacing. It chooses whom the agent answers; it is **not an access control**: a
  participant may take any display name.
- An instruction, a task's hand-back among them, never reaches a policy, so it
  is answered whoever it came from.
- `close()` closes the policy it wraps.

A runnable version is `examples/answering_some_people.py`.

## Thinking while listening

Give the channel a **thinker** too, and the agent keeps a thought while it
listens: its answer, when it is silent, to "what are you thinking about?"
(RFC §6.4). Not memory, not a summary: what it makes of what is said, what it
would say if given the turn, and whether that cannot wait.

```python
from roomkit import AIChannel, ClassifierSpeakPolicy, JevClassifier, LLMThinker

nova = AIChannel(
    "nova",
    provider=provider,
    system_prompt="You are Nova, the team's assistant. A licence costs 1,200 $ a year.",
    speak_policy=ClassifierSpeakPolicy(JevClassifier(), agent_name="Nova"),
    thinker=LLMThinker(small_fast_provider),
    think_wait=1.5,
)
```

`Thought(text, want_to_say, urgent)`: what the agent thinks, in the first
person; what it would say if given the turn, most important first, three at
most; and whether that cannot wait.

How the channel runs it:

1. **Only while listening.** On an event the policy leaves silent, the channel
   builds the event's context as for an answer (the agent's prompt, the
   conversation it may know), passes it through `BEFORE_AI_GENERATION` with
   `event.purpose == "thought"`, and hands what the hooks left to the thinker
   with the previous thought. A turn the agent answers waits for no thought.
2. **One call at a time per room.** Events that arrive during a call are thought
   about in the next one, from the latest context.
3. **Raising its hand.** The channel waits for the thought up to `think_wait`
   seconds (1.5). Back in time with something to say, the policy decides again
   on the same event, with the thought: the agent may offer on a turn it first
   listened to. Each decision fires `ON_SPEAK_DECISION`.
4. **Speaking empties it.** When the agent speaks or offers on a decided event,
   the turn's notes carry its thought, and what it wanted to say is emptied. An
   instruction (a task's hand-back) carries no thought and empties nothing. The
   thought is a model's reading of what people said, so whatever they said can
   reach it: the notes quote it, bound it (600 characters for the text, 300 per
   item) and name it as information to weigh, not instructions to follow.
5. **Failures keep it.** A thinker that fails, or runs out of its `timeout`,
   keeps the previous thought, logged.
6. **Ephemeral.** The thought is the channel's, per room, in memory; a restart
   starts empty, and so does a room the channel is attached to or detached
   from: a room that reuses an id never inherits another conversation's thought.

A thinker needs a speak policy: it thinks on the events the agent listens to.

### The thought in the policy

The policy reads the thought in `SpeakTurn.thought`. `ClassifierSpeakPolicy`
puts it in the classifier's state (`assistant_thought`) and, when the agent has
something to say, also asks:

| Name | Asks |
|------|------|
| `answers` | whether what the agent wants to say answers what the turn asks |
| `corrects` | whether it corrects, or warns about, what the turn says |

Then `compose()`: only wondered about and knowing the answer, the agent speaks
(`knows the answer`) rather than offers; not addressed, it offers (`has
something to add`) when that judgment reaches `proactivity` (0.5, lower is more
eager), half of it when the thought is urgent (`urgent`), never on urgency
alone.

### LLMThinker

`LLMThinker(provider, *, instructions=None, max_tokens=600,
reasoning_effort=None, timeout=10.0)` asks a model for the thought under a JSON
schema; the provider must support response schemas. It reads who the agent is
from the context's system prompt, the previous thought first, then the
conversation (the agent's own lines as `You:`, tool traffic and turn notes left
out). A small, fast model fits. Its default instructions are English and ask for
the conversation's language; replace them with `instructions=` (`{max}` is
replaced by three). The provider stays yours.

The thought is only as good as the model's discipline: a model that puts a
question or an offer of help in `want_to_say`, though told not to, makes the
agent offer where it should listen. Measure it on your own conversations.

### The generation hooks see the thinker

Every model call on the room's context passes `BEFORE_AI_GENERATION`, the
thinker's too: what a hook keeps from a model (a room without consent, a
redaction, a budget) holds for the thought. `AIGenerationEvent.purpose` tells
them apart, `"answer"` for the agent's turn and `"thought"` for its thinker. A
hook that blocks a thought keeps the previous one; what a hook changes in the
context is what the thinker reads. A hook meant for answers only returns
early on `purpose == "thought"`:

```python
from roomkit import HookResult, HookTrigger


@kit.hook(HookTrigger.BEFORE_AI_GENERATION)
async def no_ai_without_consent(event, ctx) -> HookResult:
    if not consented(event.room_id):
        return HookResult.block("no consent")  # no answer, no thought
    return HookResult.allow()
```

### Following the thought

Every new thought fires `ON_THOUGHT` (async), with the room, the channel, the
thought, the one it replaces, and how long the thinker call that brought it took
(`duration_ms`). A thought emptied because the agent spoke came from no call: its
`duration_ms` is `None`. A call that fails, or brings back the same thought, fires
nothing.

```python
from roomkit import HookExecution, HookTrigger, ThoughtEvent


@kit.hook(HookTrigger.ON_THOUGHT, execution=HookExecution.ASYNC)
async def on_thought(event: ThoughtEvent, ctx) -> None:
    took = f"{event.duration_ms} ms" if event.duration_ms is not None else "no call"
    logger.info("%s thinks (%s): %s", event.channel_id, took, event.thought.text)
```

`ON_THOUGHT` is the agent's thought while it listens; `ON_AI_THINKING` is a
model's reasoning during a turn.

For tests, `MockThinker([Thought(...), ...], delay=0.0, error=None)` returns
scripted thoughts in order (the last one repeats) and records each call.

A runnable version is `examples/thinking_while_listening.py`.

## A cut answer

When someone speaks over the agent's voice and cuts it off (a barge-in, RFC
§12.3.13), the voice channel records the cut: the text handed to speech, how
long it played, and the answer it cut (`metadata.answer_channel_id`,
`metadata.answer_responds_to`). The AI channel uses that record twice.

Only a record a voice channel of the room wrote counts: an `internal` event with
no participant, from a channel bound to the room as voice. Metadata is anyone's
to write; a message that claims to be a cut record is ignored.

**The context marks the answer.** In the context the agent's next turn reads,
the cut answer ends with:

```
[You were interrupted while saying this: the person may not have heard the end of it.]
```

so the model does not take for heard what was cut, and does not refer to it as
said. This holds for every AI channel, with or without a speak policy.

**The policy reads the cut.** When the event came after the cut and the agent
has not answered since, `SpeakTurn.cut` is a `CutReply(text, played_ms, at)`.
`ClassifierSpeakPolicy` then asks one more question, in the same call:

| Name | Asks |
|------|------|
| `resume` | whether the turn spoken over the agent leaves it free to go on: an acknowledgement, a thanks, a short reaction, talking over it by accident; or wants the turn: a question, a request, a correction, asking it to stop |

At 0.4 or above, unless the turn asks for quiet, the speaker is not done, or a
request for quiet still stands, the agent speaks with the reason `resume after
cut`, and its notes ask it to go on from where it was cut, with a short link
back, without repeating what was heard. Otherwise the turn decides as without a
cut: "Wait, and for Montreal?" over a forecast is answered as a question.

Which part of the answer was heard is not guessed: the record keeps the text
handed to speech and how long it played, and the model judges from those.

A runnable version, without audio, is `examples/resume_after_cut.py`.
