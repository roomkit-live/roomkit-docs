# MiniMax Provider

[MiniMax](https://platform.minimax.io) serves `MiniMax-M3.1-Flash` and `MiniMax-M3`, and publishes its weights, behind an OpenAI-compatible Chat Completions API. RoomKit's provider is a thin subclass of `OpenAIAIProvider`. Its reason to exist is a model's reasoning: MiniMax asks for it back in a tool loop, and RoomKit sends none to a generic server.

## Install

```bash
pip install roomkit[minimax]
```

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.minimax import MiniMaxAIProvider, MiniMaxConfig

provider = MiniMaxAIProvider(
    MiniMaxConfig(
        api_key="...",               # from https://platform.minimax.io
        model="MiniMax-M3.1-Flash",  # or MiniMax-M3
    )
)

kit = RoomKit()
kit.register_channel(AIChannel("assistant", provider=provider, system_prompt="You are helpful."))
```

See `examples/minimax_ai.py` for a turn with a tool, then a second turn.

## On a server of your own

The class is the choice, whatever the URL. A model served by a vLLM, a llama.cpp or a gateway gets its reasoning back the same way:

```python
provider = MiniMaxAIProvider(
    MiniMaxConfig(api_key="unused", base_url="http://gpu:8000/v1", model="MiniMaxAI/MiniMax-M3")
)
```

`OpenAIAIProvider` on that URL sends no reasoning, and has no setting for it. Run the server with its MiniMax reasoning parser, or the model's reasoning stays in its answer.

## What goes back

A tool round of a turn, with this provider and with a generic one:

```python
# MiniMaxAIProvider
{"role": "assistant", "content": "Looking.", "reasoning_content": "Why.", "tool_calls": [...]}
# OpenAIAIProvider
{"role": "assistant", "content": "Looking.", "tool_calls": [...]}
```

- A round that did not think sends no field: MiniMax documents that a model may call a tool without thinking, and no refusal.
- An answer that reasoned carries the field too.
- A later turn's request holds no reasoning, in any message: RoomKit stores none ([RFC §6.4](https://github.com/roomkit-live/roomkit-specs/blob/main/roomkit-rfc.md)).

Where the field is read:

| Where | Read from | Checked |
|---|---|---|
| MiniMax's API | `reasoning_content`, "must be kept when passing the message back" | Its documentation, not its wire |
| M2's and M3's chat templates | `reasoning_content`, else a tag in the text (`<think>` on M2, `<mm:think>` on M3) | Rendered on llama.cpp |
| llama.cpp | `reasoning_content`, an empty one treated as none | Run, build b11960 |
| vLLM | `reasoning_content`, renamed `reasoning`, both handed to the template | Read in its source and tests, not run |
| OpenRouter | `reasoning_content`, an alias of `reasoning` | Its documentation |

On llama.cpp with M2's and M3.1's templates, each request of a turn reuses the whole prompt before it, the field included. The next turn's first request drops the last turn's reasoning, and reprocesses that turn's rounds.

## Thinking

```python
MiniMaxConfig(api_key="...", model="MiniMax-M3.1-Flash", reasoning_effort="low")
```

`reasoning_effort` (`low`, `medium`, `high`, `xhigh`, `max`, default `max`) is effective on M3.1-Flash alone, on a turn with tools as on any other, and a turn's own effort outranks it. On a catalogued model, another level goes as the nearest MiniMax takes (`minimal` as `low`), and a turn that states off sends `low` on M3.1-Flash, which answers 400 to `none`, and `thinking: {"type": "disabled"}` on M3. On a model the catalogue does not name, the effort goes as stated, `none` included, and a bare off sends nothing.

`max_tokens` defaults to no cap. Reasoning counts toward it, and MiniMax documents that a cap too small yields `finish_reason: "length"` with an empty answer. A cap goes as `max_completion_tokens`, MiniMax's field for new integrations; `use_max_completion_tokens=False` for a server that reads `max_tokens` alone. `supports_custom_temperature` defaults to `False`: MiniMax's API ignores `temperature` and always uses 1, the value its model cards recommend; set it for a server of your own that honours it.

MiniMax's API documents `response_format` for `MiniMax-Text-01` alone, so `supports_response_schema` is `False` by default. Set it on the config for a server that takes it.

## Pricing

`available_models()` carries MiniMax's rates. `MiniMax-M3`'s are its standard tier: past 512k input tokens they double, which its `long_context_*` fields carry. Its `priority` tier costs 1.5 times as much, which `ModelPricing` has no place for. MiniMax documents no `/models` listing.
