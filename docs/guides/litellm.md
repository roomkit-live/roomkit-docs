# LiteLLM Proxy Provider

[LiteLLM](https://docs.litellm.ai) is best known as a Python library, but its **proxy** is a different thing: a self-hosted **AI gateway** that fronts 100+ upstream providers behind one OpenAI-compatible endpoint, adding **virtual keys, per-key budgets, rate limits, and central routing**. Enterprises deploy it so applications never hold vendor keys — they hold a gateway key with a budget.

RoomKit talks to that gateway with `LiteLLMAIProvider`, a thin subclass of `OpenAIAIProvider`: message building, tool calling, streaming, and thinking-trace surfacing all come for free. Only the reasoning parameters and model discovery differ.

!!! note "The proxy, not the SDK"
    `pip install roomkit[litellm]` installs the `openai` SDK, **not** the `litellm` package. RoomKit is already a provider abstraction with native, full-fidelity providers; running LiteLLM's in-process abstraction underneath it would add a second normalisation layer and a heavy dependency for nothing. The gateway keeps all of LiteLLM's value (keys, budgets, routing) on the server where it belongs.

## Install

```bash
pip install roomkit[litellm]
```

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.litellm import LiteLLMAIProvider, LiteLLMConfig

provider = LiteLLMAIProvider(
    LiteLLMConfig(
        api_key="sk-...",                  # virtual key or master key
        base_url="http://localhost:4000",  # your proxy; /v1 suffix also works
        model="claude-sonnet",             # public model name from the proxy's config.yaml
    )
)

kit = RoomKit()
ai = AIChannel("ai-assistant", provider=provider, system_prompt="You are helpful.")
kit.register_channel(ai)
```

The `model` field is the **public alias** the proxy's operator configured (`model_name` in the proxy's `config.yaml`) — which alias routes to which upstream is the deployment's decision, not roomkit's. Both `model` and `api_key` are required: a gateway's whole point is per-key auth and budgets (for a dev proxy running without authentication, pass any placeholder key).

## Listing models

A gateway serves whatever its operator configured, so `available_models()` is empty by design and `context_window` is `None` for any alias — the honest answer for a name roomkit cannot know offline. `list_models()` asks the proxy, which does know:

```python
models = await provider.list_models()
for m in models:
    print(m.id, m.context_window, m.supports_vision, m.pricing)
await provider.close()
```

It reads LiteLLM's **`/model/info`** endpoint rather than the barer `/v1/models`: alongside each public name it reports the context window, vision support, and per-token costs from the proxy's own cost map, mapped into `ModelInfo` (`context_window`, `supports_vision`, `pricing` with per-million rates including cache reads/writes). History trimming and budget dashboards work against the deployment's real numbers. Load-balanced model groups (several deployments under one public name) collapse to a single model.

Two honesty rules, verified against a live proxy:

- An operator-defined alias absent from LiteLLM's cost map reports **nothing** (`None` window, `None` vision, no pricing) rather than gaining invented metadata.
- LiteLLM defaults *unknown* costs to `0` rather than null, so a `0`/`0` rate pair maps to `pricing=None` — a $0 price would tell a budget dashboard the route is free while the gateway may well be billing it.

## Thinking / reasoning

LiteLLM normalises reasoning across every upstream it fronts: OpenAI's `reasoning_effort` is translated to Anthropic thinking budgets, Gemini thinking, DeepSeek, and the rest, and the trace comes back in `reasoning_content` — exactly the field RoomKit's inherited streaming reader surfaces as `StreamThinkingDelta` (rendered 💭 by `CLIChannel(show_thinking=True)`).

```python
ai = AIChannel(
    "assistant",
    provider=LiteLLMAIProvider(LiteLLMConfig(
        api_key="sk-...",
        model="claude-sonnet",
        reasoning_effort="high",   # used when thinking_budget is None
    )),
    thinking_budget=4096,          # >0 → explicit token budget; 0 → off
)
```

| `thinking_budget` | Request sent to the proxy |
|-------------------|---------------------------|
| `None` | `reasoning_effort: <configured or per-turn effort>` — omitted entirely if unset (model decides) |
| `0` | nothing — no reasoning parameters at all, the routed model's own default applies |
| `> 0` | `thinking: {type: "enabled", budget_tokens: <budget>}` — explicit budget |

Reasoning is omitted on tool-call turns, matching the OpenAI provider — the gateway fronts the same upstreams that reject it alongside function tools.

!!! warning "Why `0` sends nothing"
    LiteLLM has no disable token that survives every upstream translator — verified live against 1.79.0: its Gemini mapper rejects `reasoning_effort="none"` with a 500, and its Anthropic mapper rejects both `"none"` and `"disable"`. Any explicit spelling would break on some routes while working on others, so omission is the one portable behaviour. To force thinking off for an alias, state it where the upstream is known: per-model in the proxy's `config.yaml`, or via `extra_body` with the token that route's translator accepts (e.g. `extra_body={"reasoning_effort": "disable"}` for a Gemini route).

## Trying it without an upstream key

The proxy can mock responses, so the whole path is testable with no vendor account:

```yaml
# litellm-config.yaml
general_settings:
  master_key: sk-litellm-test
model_list:
  - model_name: mock-model
    litellm_params:
      model: openai/mock
      api_key: fake-upstream-key
      mock_response: "Hello from the gateway!"
```

```bash
uvx --from 'litellm[proxy]' litellm --config litellm-config.yaml --port 4000
LITELLM_API_KEY=sk-litellm-test python examples/litellm_ai.py --model mock-model
```

See `examples/litellm_ai.py` for an interactive chat and a `--list-models` mode.

## How it works

| Aspect | Behaviour |
|--------|-----------|
| Transport | OpenAI Chat Completions API at your proxy's URL (with or without `/v1`) |
| Config | `LiteLLMConfig` **subclasses** `OpenAIConfig`, inheriting every request field (`temperature`, `reasoning_effort`, `include_stream_usage`, `extra_body`, …) so the two never drift |
| Streaming & tools | Inherited unchanged from `OpenAIAIProvider`, including `reasoning_content` thinking handling |
| Vision | `supports_vision` is `True`: whether a routed model reads images is the gateway's call, so images pass through and a text-only route answers with an error |
| Telemetry | Errors and TTFB metrics are tagged `provider="litellm"` |
