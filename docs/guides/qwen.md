# Qwen Provider

[Alibaba Cloud Model Studio](https://www.alibabacloud.com/help/en/model-studio/) serves the **Qwen** lineup behind an OpenAI-compatible Chat Completions API — `qwen3.7-max`, `qwen3.7-plus`, `qwen3.6-flash`, `qwen3-coder-plus`, `qwen3-vl-plus` — with 1M-token context windows on most of the line and image input on all but the flagship.

Because Model Studio speaks the OpenAI API verbatim, RoomKit's provider is a thin subclass of `OpenAIAIProvider`: message building, tool calling and streaming all come for free. Four things are Qwen's own, and they are what the subclass exists for.

## Install

```bash
pip install roomkit[qwen-ai]
```

The extra is `qwen-ai`, not `qwen` — `qwen-tts` and `qwen-asr` are separate voice extras installing different packages.

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.qwen import QwenAIProvider, QwenConfig

provider = QwenAIProvider(
    QwenConfig(
        api_key="sk-...",        # from https://bailian.console.aliyun.com
        model="qwen3.7-max",
    )
)

kit = RoomKit()
ai = AIChannel("ai-assistant", provider=provider, system_prompt="You are helpful.")
kit.register_channel(ai)
```

See `examples/qwen_ai.py` for an interactive chat.

## Choosing your endpoint

Model Studio runs several deployments and your key belongs to exactly one. The default is the international endpoint; everything else is a `base_url` override:

| Deployment | `base_url` |
|---|---|
| International (default) | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| China (Beijing) | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| US (Virginia) | `https://dashscope-us.aliyuncs.com/compatible-mode/v1` |
| Workspace-scoped, any region | `https://{WorkspaceId}.{region}.maas.aliyuncs.com/compatible-mode/v1` |

The workspace form takes its id from the Model Studio console, which is why it cannot be a default — there is no correct value to guess.

## Thinking

Qwen takes a boolean switch plus a **token cap**, both outside the OpenAI schema:

```python
QwenConfig(
    api_key="sk-...",
    model="qwen3.7-plus",
    enable_thinking=True,   # None leaves the model's own default
)
```

Per turn, `AIChannel(thinking_budget=2048)` maps **straight onto** Qwen's `thinking_budget` — this is the one provider where RoomKit's budget is the vendor's parameter rather than an approximation of it. A budget of `0` switches thinking off. The trace comes back in a dedicated field and surfaces as `AIResponse.thinking` (and as `StreamThinkingDelta` while streaming).

!!! note "`reasoning_effort` is not sent"
    The field is inherited from `OpenAIConfig` and deliberately ignored here. Model Studio accepts it only for the third-party DeepSeek models it also hosts — see the [DeepSeek guide](deepseek.md) for talking to those directly.

## Model discovery

Model Studio's OpenAI-compatible deployment serves `/chat/completions` **and nothing else** — there is no `/v1/models`. `list_models()` therefore returns RoomKit's offline catalog rather than the account's own set, which is also why that catalog carries every id a caller is expected to reach:

```python
for m in await provider.list_models():
    print(m.id, m.context_window, m.supports_vision)
```

An id outside the catalog still works on the wire; it just reports `context_window is None`, and `supports_vision` defaults to `True` so an image is never silently dropped.

## Scope: Qwen, not the gateway

Model Studio also hosts DeepSeek, Kimi, GLM and MiniMax on the same endpoint. This provider is scoped to the Qwen family it is named for — its catalog, prices and thinking switch are Qwen's. Reaching the others through it would work on the wire and report the wrong metadata for every one of them; use their own providers, or `OpenRouterAIProvider` if a single key across vendors is what you want.

## Pricing

The catalog carries Alibaba's **international list prices**, which is what `ModelPricing.price()` costs a response against.

!!! note "Promotions, regions and tiers"
    Alibaba runs near-permanent limited-time promotions, quotes different rates per deployment (Beijing bills `qwen3.7-max` roughly a third below international), and tiers several models by input length. `ModelPricing` carries one list rate and a single long-context threshold, so `qwen3-coder-plus` and `qwen3-vl-plus` — four and three bands respectively — report **no** price rather than one that would understate a long-context bill by up to 12x. Check the [billing page](https://www.alibabacloud.com/help/en/model-studio/billing-for-model-studio) when the figure has to be exact.
