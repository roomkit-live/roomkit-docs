# Kimi Provider

[Kimi](https://platform.kimi.ai) serves `kimi-k3.1-pro`, `kimi-k3.1-pro-highspeed` and `kimi-k3`, and publishes its weights, behind an OpenAI-compatible Chat Completions API. RoomKit's provider is a thin subclass of `OpenAIAIProvider`. Its reason to exist is a model's reasoning: Kimi requires it back on a tool round, and RoomKit sends none to a generic server.

## Install

```bash
pip install roomkit[kimi]
```

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.kimi import KimiAIProvider, KimiConfig

provider = KimiAIProvider(
    KimiConfig(
        api_key="...",           # from https://platform.kimi.ai
        model="kimi-k3.1-pro",   # or kimi-k3.1-pro-highspeed, kimi-k3
    )
)

kit = RoomKit()
kit.register_channel(AIChannel("assistant", provider=provider, system_prompt="You are helpful."))
```

See `examples/kimi_ai.py` for a turn with a tool, then a second turn.

## On a server of your own

The class is the choice, whatever the URL. A model served by a vLLM or a gateway gets its reasoning back the same way:

```python
provider = KimiAIProvider(
    KimiConfig(api_key="unused", base_url="http://gpu:8000/v1", model="moonshotai/Kimi-K3")
)
```

`OpenAIAIProvider` on that URL sends no reasoning, and has no setting for it. Kimi's own rules stay on its own endpoint: behind another URL the server decides the tool names, and what a model the catalogue does not name reads. `supports_custom_temperature=True` sends a temperature to a server that honours it.

## What goes back

A tool round of a turn, with this provider and with a generic one:

```python
# KimiAIProvider
{"role": "assistant", "content": "Looking.", "reasoning_content": "Why.", "tool_calls": [...]}
# OpenAIAIProvider
{"role": "assistant", "content": "Looking.", "tool_calls": [...]}
```

Kimi answers 400 to a tool round without the field ("thinking is enabled but reasoning_content is missing in assistant tool call message", its error reference), and K3 always thinks. So every round that called tools carries it, and a round with no reasoning carries a single space:

```python
{"role": "assistant", "content": None, "reasoning_content": " ", "tool_calls": [...]}
```

That is every round of an earlier turn, since RoomKit stores no reasoning ([RFC §6.4](https://github.com/roomkit-live/roomkit-specs/blob/main/roomkit-rfc.md)). An empty field reads as a missing one: LiteLLM's Moonshot code sends the same space, as "the minimum value the API accepts". None of this was checked on Kimi's wire; `examples/kimi_ai.py` does it, with a key.

!!! warning "Earlier turns"
    Kimi's page says K3 requires the reasoning "in multi-turn conversations and tool-call loops", and its API reference that without it "the model may lose reasoning context". RoomKit sends it back for as long as a turn lasts, not in a later turn. Its answers go without the field, for which Kimi documents no refusal.

Where the field is read:

| Where | Read from | Checked |
|---|---|---|
| Kimi's API | `reasoning_content` | Its documentation, not its wire |
| K2.6's and K2.7's chat templates | `reasoning`, else `reasoning_content` | Rendered on llama.cpp |
| K3's renderer (`encoding_k3.py`) | `reasoning_content`, else `reasoning` | Read in its source |
| llama.cpp | `reasoning_content`, an empty one treated as none | Run, build b11960 |
| vLLM | `reasoning_content`, renamed `reasoning`, both handed to the template | Read in its source and tests, not run |
| OpenRouter | `reasoning_content`, an alias of `reasoning` | Its documentation |

On llama.cpp with K2.6's and K2.7's templates, each request of a turn reuses the whole prompt before it, the field included. A single space renders as `<think> </think>`, one token more than an empty one. The next turn's first request drops the last turn's reasoning, and reprocesses that turn's rounds.

## Thinking

```python
KimiConfig(api_key="...", model="kimi-k3.1-pro", reasoning_effort="low")
```

`reasoning_effort` (`low`, `high`, `max`, default `max`) goes on a turn with tools as on any other, and a turn's own effort outranks it. K3 always thinks, so a turn that states off sends `low` for the catalogued models. On a model the catalogue does not name, the effort goes as stated, and a bare off sends nothing.

`max_tokens` defaults to no cap: reasoning counts toward it, and Kimi recommends at least 16,000. `supports_custom_temperature` defaults to `False`, since K3 fixes `temperature` and asks that it be left out.

## Images

K3 reads images, and Kimi's API takes them as base64 or as `ms://<file-id>`, not as a public URL. An `AIImagePart` carrying a URL is refused by the vendor (`unsupported image url`).

## Tool names

On Kimi's own endpoint a tool name must match `[a-zA-Z_][a-zA-Z0-9_-]{0,127}`, Kimi's documented rule, checked before the request.

## Pricing

`available_models()` carries Kimi's rates. `cache_write_per_million` is the 5-minute write, its default tier; a 1-hour write costs twice as much, which `ModelPricing` has no place for. The K2 series is left out: closed to new users, its retirement planned for late December 2026. An id out of the catalogue still works.
