# Classifiers

Some decisions need understanding that code does not have: was the agent
addressed, did the person finish, which language is this. A **classifier**
answers such narrow, typed questions about a state with probabilities, not
generated text (RFC §6.8). You ask your questions together, in one call, and
compose the answers in code, where each judgment stays visible and measurable.

```python
from roomkit import ChoiceQuestion, JevClassifier, ScoreQuestion, YesNoQuestion

questions = {
    "addressee": ChoiceQuestion(
        "Whom is the last message addressed to?",
        {
            "nova": "Nova, the assistant: she is named, or the request is plainly hers",
            "person": "a person in the room, named or plainly meant",
            "nobody": "nobody in particular: the room, or whoever knows",
        },
    ),
    "asks": YesNoQuestion(
        "Does the last message ask for something?",
        no="a statement, a reaction, or thinking aloud",
    ),
    "urgency": ScoreQuestion(
        "How urgent is what the last message asks for?",
        ("nothing asked, or it can wait", "within the day", "right now"),
    ),
}

classifier = JevClassifier()  # reads TYPESAFE_API_KEY
state = {"agent": "Nova", "people": ["Sylvain", "Paul"],
         "last": {"from": "Paul", "text": "Nova, can you sum up?"}}
answers = await classifier.classify(state, questions)

if answers.choice("addressee") == "nova":
    ...  # hers
elif answers.yes("asks") >= 0.5 and answers.score("urgency") >= 1.5:
    ...  # urgent, and nobody was asked
```

A runnable version, on five turns of a meeting, is
`examples/classifier_judgments.py`.

## Questions

| Question | Asks | Answer |
|----------|------|--------|
| `YesNoQuestion(instructions, yes=None, no=None)` | whether something holds; `yes` and `no` say what each covers, when that needs saying | `YesNoAnswer.probability`: of yes, in [0, 1] |
| `ChoiceQuestion(instructions, options)` | one option among several, each described (two at least) | `ChoiceAnswer.choice`, the most probable, and `probabilities` for every option |
| `ScoreQuestion(instructions, levels)` | a degree on an ordered scale, lowest level first, each described (two at least) | `ScoreAnswer.score`, the expected level (0 for the first, a fraction between two), and `probabilities` per level |

The question names are for your code; the wording is what the classifier reads,
so put the whole meaning in `instructions` and the options or levels. Ask one
narrow judgment per question: three small questions composed in code are easier
to check, and to correct, than one question that decides everything.

## State

The state is what the questions are about: a text, or JSON-compatible data. Name
its parts (`"last"`, `"recent"`, `"people"`) and refer to them in the questions:
"Whom is the last message addressed to?" reads `last` in the light of the rest.

## Answers

`classify()` returns an `Answers`, a dict of answers by question name, with typed
reads:

```python
answers.yes("asks")          # float: the probability of yes
answers.choice("addressee")  # str: the chosen option
answers.score("urgency")     # float: the expected level
answers["addressee"].probabilities  # {"nova": 0.0, "person": 0.01, "nobody": 0.99}
```

A read of the wrong kind raises `TypeError`. A classifier answers every question
or none: anything that keeps it from answering (a refused request, an answer it
cannot read, the end of its wait) raises `ClassifierError`. Decide in your code
what a failure means; a speak policy, for instance, lets the agent speak.

## Implementations

| Classifier | Probabilities | Latency | Needs |
|------------|---------------|---------|-------|
| `JevClassifier(api_key=None, *, model=None, timeout=3.0, client=None)` | calibrated: threshold on them, read a spread as doubt | ~150 ms for the whole call | `pip install roomkit[typesafe]`, `TYPESAFE_API_KEY` |
| `OpenAIClassifier(api_key=None, *, model="gpt-6-luna", timeout=3.0, client=None)` | probabilities from OpenAI's Decisions API, often near 0 or 1; OpenAI advises calibrating thresholds on labelled data | ~150–450 ms for the whole call | `pip install roomkit[openai]`, `OPENAI_API_KEY` |
| `SystemOneClassifier(base_url=None, *, model=None, api_key=None, timeout=10.0, client=None)` | the decision model's own, through any server with a System One API (below) | from ~50 ms on a local GPU to ~800 ms, by server and model | `pip install roomkit[typesafe]`, the server |
| `LlamaCppClassifier(config, *, timeout=30.0)` | the decision model's own; RoomKit runs `llama-server` itself | ~50 ms per call on a GPU with d1 | `pip install roomkit[typesafe,llamacpp]` |
| `LLMClassifier(provider, *, timeout=10.0, max_tokens=1000)` | 0 or 1: the model picks one answer per question | a generation (about 1 s on Claude Haiku 5.5) | an AI provider whose `supports_response_schema` is true |
| `MockClassifier(answers=None, *, error=None)` | as scripted | none | — |

**Jev** is TypeSafe's System One model, trained to answer typed questions with
calibrated probabilities. Every question of a call is answered in parallel, so a
dozen questions cost about what one does. Without a `client`, it makes an
`AsyncTypeSafeClient` and closes it on `close()`; a client you pass stays yours.
The `model` you give goes with every call, through a client you pass too.

**`SystemOneClassifier`** asks a decision model on any server that speaks the
System One API (`POST {base_url}/v1/systemone`), through the TypeSafe SDK. The
API is TypeSafe's, and other servers have taken it up, open and hosted alike:

| Server | `base_url` | Models | Measured (three questions, one call) |
|--------|------------|--------|--------------------------------------|
| TypeSafe | `https://api.typesafe.ai` | `jev-latest` | 78–125 ms |
| LiteLLM proxy ≥ 1.104.2 | your proxy | the names in its config: Jev, OpenAI's Decisions models (which it translates), Cloudflare Clef, Databricks, Perplexity… | 72–101 ms to Jev, 175–815 ms to `gpt-6-luna` |
| Ollama ≥ 0.35 | `http://localhost:11434` | `d3:2b`, `d1:2b`, `nimble`, `tev1`, `clef`… | 58–73 ms with `d3:2b`, 250–360 ms with `d1:2b` (on Linux it needs Ollama's MLX add-on) |
| llama.cpp `llama-server` | its address | a decision model in GGUF, LiquidAI's d1 among them (or `LlamaCppClassifier`, below) | 43–66 ms with d1 |
| vLLM ≥ 0.32, vLLM Semantic Router | its address | the `vllm-sr/d3` family | not measured |

The local times are on an RTX 4070.

`base_url` is the server's root, and `model` is the name the server gives the
model. `api_key` is only for a server that asks for one; a local server asks
for none, and the classifier never reads `TYPESAFE_API_KEY`, which is another
server's.

A score is the expected level, computed from each level's probability: some
servers put the likeliest level in their `score` field instead (llama.cpp did).
That gives a score the same meaning whatever the server.

The models differ in quality and in licence. LiquidAI's d1 comes under the LFM
Open License, which allows commercial use only below 10 million USD of annual
revenue. The `vllm-sr/d3` family is Apache-2.0.

With the decision rule of `examples/classifier_judgments.py`, four test turns
(two of the example's, an urgent call to the room, a thanks) came out 4/4 with
`gpt-6-luna`, 3/4 with Jev and `d3:2b`, and 2/4 with d1. On the example's own
five turns, d1 takes the expected decision every time, twice for another
reason. Measure a model on your own turns before you rely on it.

```python
from roomkit import SystemOneClassifier

local = SystemOneClassifier("http://localhost:11434", model="d3:2b")
proxy = SystemOneClassifier("http://localhost:4000", model="jev", api_key="sk-...")
```

The model is required: without it the TypeSafe SDK would send its own
default, a TypeSafe model another server does not have. A `client` you pass
sends its own model when you give none.

**`LlamaCppClassifier`** runs a decision model itself through llama.cpp, the
way [`LlamaCppAIProvider`](llamacpp.md) runs a chat model. The first question,
or `await classifier.start()`, downloads the pinned llama.cpp build and the
model, and starts `llama-server` on a free local port. Then it asks the
server's System One API. `close()` stops the server. A server that cannot
start is a `ClassifierError`. The first question waits for that start, which
`LlamaCppConfig.startup_timeout` bounds, not `timeout`: call `start()` before
the first turn to keep the download off it.

```python
from roomkit import LlamaCppClassifier
from roomkit.providers.llamacpp import LlamaCppConfig

classifier = LlamaCppClassifier(LlamaCppConfig(model="LiquidAI/d1-2B-GGUF:Q8_0"))
```

`LlamaCppConfig` is the provider's, and `max_tokens` and `enable_thinking` do
not apply. Some decision models read their whole prompt in one batch: vllm-d3,
laya, clef and d1-omni. Give them a batch that holds it, for example
`extra_args=["-b", "8192", "-ub", "8192"]`.

**`OpenAIClassifier`** answers on OpenAI's Decisions API (`POST /v1/decisions`,
public beta, `gpt-6-luna`), which takes the same three kinds of question: a
yes/no question goes as a `predicate` (what a yes and a no cover join its
instructions, the API taking instructions alone), a choice as a `choice`, a
score as a `score` over labelled levels. The API reads a text, so a structured
state is sent as its JSON. A question the API declines to answer (a `refusal`)
fails the whole call with `ClassifierError`. The OpenAI SDK 2.x has no
`decisions` resource yet: the call goes through the client's generic `post`,
with its key, retries and errors. Without a `client`, it makes an `AsyncOpenAI`
and closes it on `close()`.

On the questions of a discussion's dispatch policy (eighteen messages to a team
of four), it picked right 17 times and Jev 18; its probabilities sit nearer 0
and 1. A policy whose thresholds were measured on Jev (the speak policy) may
read them differently: measure it on your own turns before relying on it.

**`LLMClassifier`** asks any AI provider for one JSON document under a response
schema (RFC §6.7): a boolean per yes/no question, an option per choice, a level
per score. Use it where a generative model is what you have. Its probabilities
are not calibrated, so a threshold between 0 and 1 means nothing; and a
generation is slower than a trained classifier. `close()` leaves the provider
open: it is yours, and often shared with an AI channel.

**`MockClassifier`** answers from a script by question name: a float for a
yes/no question, an option for a choice, a level for a score, or an answer
object. A question the script does not name gets no, the first option, or the
lowest level. Given a list of scripts, the calls take them in order and the last
one repeats. `calls` records every state and question set; `error=` makes every
call raise.

```python
from roomkit import MockClassifier

classifier = MockClassifier({"asks": 0.9, "addressee": "nobody", "urgency": 2})
```

## Writing a classifier

Subclass `Classifier` and implement `classify()`; `close()` is optional. Return
an answer for every question, of its kind, or raise `ClassifierError`;
`roomkit.classifiers.base.check_answers(questions, answers)` checks both. Bound
your wait, and document how far your probabilities can be trusted.

```python
from roomkit import Answers, Classifier
from roomkit.classifiers import State, check_answers


class MyClassifier(Classifier):
    async def classify(self, state: State, questions) -> Answers:
        answers = Answers()
        ...  # one answer per question
        return check_answers(questions, answers)
```
