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
         "last": {"from": "Paul", "text": "Nova, tu peux résumer ?"}}
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
| `LLMClassifier(provider, *, timeout=10.0, max_tokens=1000)` | 0 or 1: the model picks one answer per question | a generation (about 1 s on Claude Haiku 4.5) | an AI provider whose `supports_response_schema` is true |
| `MockClassifier(answers=None, *, error=None)` | as scripted | none | — |

**Jev** is TypeSafe's System One model, trained to answer typed questions with
calibrated probabilities. Every question of a call is answered in parallel, so a
dozen questions cost about what one does. Without a `client`, it makes an
`AsyncTypeSafeClient` and closes it on `close()`; a client you pass stays yours.

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
from roomkit import Classifier
from roomkit.classifiers.base import Answers, State, check_answers


class MyClassifier(Classifier):
    async def classify(self, state: State, questions) -> Answers:
        answers = Answers()
        ...  # one answer per question
        return check_answers(questions, answers)
```
