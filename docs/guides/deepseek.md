# DeepSeek Provider

[DeepSeek](https://platform.deepseek.com) serves two models — `deepseek-v4-pro` and `deepseek-v4-flash` — behind an OpenAI-compatible Chat Completions API. Both are tool-capable, carry a 1M-token context window, and think by default.

Because DeepSeek speaks the OpenAI API verbatim, RoomKit's provider is a thin subclass of `OpenAIAIProvider`: message building, tool calling, streaming and model discovery all come for free. Three things are DeepSeek's own, and they are what the subclass exists for.

## Install

```bash
pip install roomkit[deepseek]
```

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.deepseek import DeepSeekAIProvider, DeepSeekConfig

provider = DeepSeekAIProvider(
    DeepSeekConfig(
        api_key="sk-...",            # from https://platform.deepseek.com
        model="deepseek-v4-pro",     # or deepseek-v4-flash
    )
)

kit = RoomKit()
ai = AIChannel("ai-assistant", provider=provider, system_prompt="You are helpful.")
kit.register_channel(ai)
```

See `examples/deepseek_ai.py` for an interactive chat.

## Thinking

DeepSeek takes reasoning as a **nested `thinking` object**, not OpenAI's top-level `reasoning_effort` string — a top-level field is silently ignored, so the provider builds the nested form for you:

```python
DeepSeekConfig(
    api_key="sk-...",
    model="deepseek-v4-pro",
    reasoning_effort="high",   # "low" | "high" | "max", nested under `thinking`
    enable_thinking=True,      # None leaves DeepSeek's default, which is on
)
```

Per turn, `AIChannel(thinking_budget=...)` overrides the config: `0` switches thinking off, any positive value switches it on.

!!! warning "Token budgets are ignored by this API"
    DeepSeek accepts no reasoning token cap — `reasoning_effort` is the only lever over how long the model thinks. RoomKit deliberately drops the *size* of `thinking_budget` rather than inventing a budget-to-effort mapping DeepSeek never published; only its sign is read.

The reasoning trace comes back in a dedicated field and surfaces as `AIResponse.thinking` (and as `StreamThinkingDelta` while streaming).

## Prompt caching and cost

DeepSeek's context cache is **automatic** — there is nothing to mark, and populating it is free. Cache hits are billed at roughly 2% of the uncached input rate, and the provider maps DeepSeek's own `prompt_cache_hit_tokens` / `prompt_cache_miss_tokens` counters onto RoomKit's canonical `cache_read_input_tokens` and `input_tokens`, so a cost dashboard prices them at the cache rate instead of full input.

```python
response = await provider.generate(context)
print(response.usage)
# {'input_tokens': 40, 'output_tokens': 50, 'cache_read_input_tokens': 960}
```

## No images

DeepSeek's API is text-only. `provider.supports_vision` is `False` for every catalogued model, so `AIImagePart` content is filtered out before it reaches the wire rather than rejected by the endpoint.

## Why not the Anthropic-compatible endpoint

DeepSeek also fronts an Anthropic-shaped API at `https://api.deepseek.com/anthropic`, which is what tools built for Claude point at. RoomKit uses the OpenAI-shaped one because it is the richer of the two: the Anthropic path drops `cache_control` prompt caching, image content, and the models listing, and ignores `budget_tokens`. Nothing stops you from using the other — `AnthropicConfig(base_url="https://api.deepseek.com/anthropic", enable_prompt_caching=False)` works — but you get less.

## Pricing

`available_models()` carries DeepSeek's own per-million rates so `ModelPricing.price()` can cost a response offline.

!!! note "Peak / off-peak billing"
    From 2026-08-16 16:00 UTC DeepSeek bills peak and off-peak rates (peak 01:00-04:00 and 06:00-10:00 UTC, off-peak at exactly half). `ModelPricing` carries one rate per model with no time dimension, so the catalog states the **peak** column — the undiscounted one, so an off-peak call bills less than quoted rather than more. Treat the figure as indicative and check [DeepSeek's pricing page](https://api-docs.deepseek.com/quick_start/pricing) when it has to be exact.
