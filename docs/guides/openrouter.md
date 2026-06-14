# OpenRouter Provider

[OpenRouter](https://openrouter.ai) is an aggregator that exposes **300+ models** from Anthropic, OpenAI, Google, Meta, DeepSeek, Mistral, xAI, Qwen, and more — all behind a **single API key** and the OpenAI-compatible Chat Completions API. Use it to switch models (or providers) by changing one slug, without managing a key per vendor.

Because OpenRouter speaks the OpenAI API verbatim, RoomKit's provider is a thin subclass of `OpenAIAIProvider`: message building, tool calling, streaming, and reasoning all come for free. Only the routing endpoint, app-attribution headers, and model discovery differ.

## Install

```bash
pip install roomkit[openrouter]
```

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.openrouter import OpenRouterAIProvider, OpenRouterConfig

provider = OpenRouterAIProvider(
    OpenRouterConfig(
        api_key="sk-or-...",                     # from https://openrouter.ai/keys
        model="anthropic/claude-sonnet-4.5",     # any OpenRouter slug
    )
)

kit = RoomKit()
ai = AIChannel("ai-assistant", provider=provider, system_prompt="You are helpful.")
kit.register_channel(ai)
```

The `model` field is **required** — choosing the model is the whole point of OpenRouter. Pass any slug from the [models list](https://openrouter.ai/models), e.g. `openai/gpt-5.5`, `google/gemini-3.5-flash`, `deepseek/deepseek-v4-pro`, or `x-ai/grok-4.20`.

## App attribution (optional)

OpenRouter can surface your app on its leaderboards and analytics. Set `site_url` (sent as the `HTTP-Referer` header) and `app_name` (sent as `X-Title`):

```python
OpenRouterConfig(
    api_key="sk-or-...",
    model="openai/gpt-5.5",
    site_url="https://github.com/your-org/your-app",  # creates the app page
    app_name="My RoomKit App",                        # display name in rankings
)
```

Both are omitted from requests when left unset. `X-Title` only creates an app page when paired with `site_url`.

## Listing models

OpenRouter's catalog is large and moves fast, so model discovery is a first-class feature. Two entry points, inherited from every RoomKit AI provider:

```python
# Offline, curated snapshot — a classmethod, so no key, network, or SDK needed.
for m in OpenRouterAIProvider.available_models():
    print(m.id, m.display_name, m.context_window, m.supports_vision)

# Live — every model OpenRouter exposes right now, with rich metadata.
provider = OpenRouterAIProvider(OpenRouterConfig(api_key="sk-or-...", model="openai/gpt-5.5"))
models = await provider.list_models()          # 300+ ModelInfo entries
print(len(models))
for m in models:
    print(m.id, m.context_window, m.supports_vision)
await provider.close()
```

- **`available_models()`** returns a small, hand-maintained slice of current flagships (sourced from the live endpoint). Use it to discover sensible defaults without a key.
- **`list_models()`** is **authoritative**: it reads OpenRouter's `/models` endpoint and maps each entry's id, display name, `context_length`, and vision support (from `architecture.input_modalities`). Curated metadata backfills anything the endpoint leaves blank.

!!! note "Why `list_models()` reads raw JSON"
    OpenRouter's `/models` items omit the `object`/`owned_by` fields that the OpenAI SDK's `Model` type requires, so the standard `client.models.list()` cannot parse them. The provider fetches the raw JSON instead — you don't need to do anything, it just works.

See `examples/list_models.py` for a runnable comparison across providers, and `examples/openrouter_ai.py` for an interactive chat.

## Thinking / reasoning

OpenRouter exposes a **unified `reasoning` parameter** that normalises thinking tokens across every upstream provider, so a reasoning trace comes back whether you route to Claude, GPT, Gemini, or DeepSeek. The provider requests it automatically based on the channel's `thinking_budget` and the config's `reasoning_effort`:

```python
from roomkit import AIChannel

ai = AIChannel(
    "assistant",
    provider=OpenRouterAIProvider(OpenRouterConfig(
        api_key="sk-or-...",
        model="anthropic/claude-sonnet-4.5",
        reasoning_effort="high",     # used when thinking_budget is None
    )),
    thinking_budget=4096,            # >0 → reasoning on (max_tokens cap); 0 → off
)
```

| `thinking_budget` | Request sent to OpenRouter |
|-------------------|----------------------------|
| `None` | `reasoning: {effort: <reasoning_effort>}` — omitted entirely if `reasoning_effort` is unset (model decides) |
| `0` | `reasoning: {enabled: false}` — thinking off |
| `> 0` | `reasoning: {max_tokens: <budget>}` — Anthropic-style cap |

The reasoning trace streams as `StreamThinkingDelta` events, so a `CLIChannel(show_thinking=True)` renders it inline (💭) above the answer — see `examples/openrouter_ai.py`. Reasoning is omitted on tool-call turns (some models reject it alongside function tools).

!!! note "Streaming vs non-streaming"
    The thinking trace is captured on the **streaming** path (the AIChannel default). A non-streaming `generate()` call still *requests* reasoning, but OpenRouter's `message.reasoning` field is not folded into `AIResponse.thinking`.

## How it works

| Aspect | Behaviour |
|--------|-----------|
| Transport | OpenAI Chat Completions API at `https://openrouter.ai/api/v1` |
| Config | `OpenRouterConfig` **subclasses** `OpenAIConfig`, inheriting every request field (`temperature`, `reasoning_effort`, `include_stream_usage`, `use_max_completion_tokens`, …) so the two never drift |
| Streaming & tools | Inherited unchanged from `OpenAIAIProvider`, including `<think>` / `reasoning_content` handling |
| Telemetry | Errors and TTFB metrics are tagged `provider="openrouter"` |

To target a self-hosted OpenRouter-compatible proxy, override `base_url`.
