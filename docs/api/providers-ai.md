# AI Providers

::: roomkit.providers.ai.base.AIProvider

::: roomkit.providers.ai.base.AIContext

::: roomkit.providers.ai.base.AIMessage

::: roomkit.providers.ai.base.AITextPart

::: roomkit.providers.ai.base.AIImagePart

::: roomkit.providers.ai.base.AITool

::: roomkit.providers.ai.base.AIToolCall

::: roomkit.providers.ai.base.AIResponse

::: roomkit.providers.ai.base.ModelInfo

::: roomkit.providers.ai.base.ModelPricing

`ModelPricing` is also exported from the package root. Its rates are finite and
non-negative, its multipliers finite and positive, and `cost_for()` accepts only
non-negative integer token counters. Invalid accounting input raises instead of
producing a negative or non-finite cost.

::: roomkit.providers.ai.mock.MockAIProvider

## Model capability tags

A conversational provider's `list_models()` also lists the speech models its
vendor serves. They carry `transcription` (speech-to-text) or `speech`
(text-to-speech) in `ModelInfo.capabilities`, read from what the vendor
reports or from the model's name, so a model picker can keep them out of a
chat list. A curated catalog's own `capabilities` stay out of a live listing.

::: roomkit.providers.ai.model_tags

## Response schema

`AIContext.response_schema` constrains the answer of `generate()` to a JSON
Schema within a portable subset. See the
[Structured Output guide](../guides/structured-output.md).

::: roomkit.providers.ai.json_schema.check_portable_schema

::: roomkit.providers.ai.json_schema.schema_mismatch

::: roomkit.providers.ai.response_schema.ResponseSchemaError

## Image parts

Every provider turns an `AIImagePart` into what its API takes through one
reader: media type from the header, then the part's `mime_type`, then
`image/png`; a payload an encoder wrapped or left unpadded is repaired; a
corrupt one is refused before the request leaves, as a non-retryable
`ProviderError` that names the cause.

::: roomkit.providers.ai.image_parts.image_part_payload

::: roomkit.providers.ai.image_parts.image_part_base64

::: roomkit.providers.ai.image_parts.image_part_uri

## Handing the loop a tool call

Every provider hands the tool loop the same thing for the same call (RFC
§6.4): arguments as a mapping, never an error; an id no other call of the
response carries; and `partial=True` on a call whose arguments do not read as
an object, which the loop answers without running it, whatever stop reason
the response gave. The model reads why: `garbled=True` says the model wrote
the arguments unreadable; a partial call that is not garbled was cut (the
output cap, a content filter, a stream that stopped without a stop reason).
A custom provider builds its calls through these rules rather than its own.
A realtime provider hands `on_tool_call` the result of `readable_arguments`:
the mapping, or the model's text for a call whose arguments do not read, which
the channel refuses (RFC §12.4).

::: roomkit.providers.ai.tool_calls.tool_arguments

::: roomkit.providers.ai.tool_calls.readable_arguments

::: roomkit.providers.ai.tool_calls.unreadable_arguments

::: roomkit.providers.ai.tool_calls.call_cut

::: roomkit.providers.ai.tool_calls.call_garbled

::: roomkit.providers.ai.tool_calls.CallIds

::: roomkit.providers.ai.tool_calls.partial_call_error

::: roomkit.providers.ai.base.tool_call_of

## Declaring tools

A provider declares the turn's tools in its vendor's format (RFC §6.7). A
tool without parameters is declared as an object with none, which every
vendor accepts: Anthropic refuses an empty map, and Anthropic and Mistral a
declaration without a schema. Tool names differ by vendor, so a provider
checks its vendor's rule before the request and raises a `ProviderError`
naming the tool and the rule; a provider in front of a server it does not
know (a custom URL: `base_url`, Mistral's `server_url`) checks none.

::: roomkit.providers.ai.tool_declaration.declared_parameters

::: roomkit.providers.ai.tool_declaration.chat_tool_declarations

::: roomkit.providers.ai.tool_declaration.ToolNameRule

## Rendering a conversation as Chat Completions messages

OpenAI and its derivatives, Mistral and PolarGrid render a conversation
through one builder. What a provider renders differently is its
`ChatDialect`: where a model's earlier reasoning goes (inline `<think>`
tags, a message field such as Cerebras's `reasoning`, or nowhere), whether a
tool message names its tool, whether text goes flat. A derivative of
`OpenAIAIProvider` whose service renders differently sets `_chat_dialect`.

::: roomkit.providers.ai.chat_request.ChatDialect

::: roomkit.providers.ai.chat_request.chat_messages

::: roomkit.providers.ai.tool_calls.is_truncation

## Explicit models and modern request profiling

`model=` is required by both `OpenAIConfig` and `AnthropicConfig`, so a RoomKit
upgrade cannot silently change cost, latency, or model behavior. For the model
the caller selects, `OpenAIConfig` profiles first-party `gpt-5`/o-series models
to use `max_completion_tokens` and omit a custom temperature.
`AnthropicConfig` profiles modern Claude reasoning models to use adaptive
thinking and omit temperature. This prevents selected modern model ids from
being paired with parameters those models reject.

Explicit compatibility flags always win. A custom `base_url` also keeps the
conservative legacy defaults, since OpenAI-compatible and Anthropic-compatible
proxies may implement the older request shape.

The OpenAI provider uses Chat Completions. On OpenAI's own endpoint, what a
turn carrying function tools sends of `reasoning_effort` comes from the model
catalogue: the turn's effort for the reasoning models before GPT-5.4, and
`"none"` explicitly from GPT-5.4 on, where it is the only value accepted
alongside tools and leaving it out answers 400 on the models that default
higher. A model the catalogue does not tag, a custom `base_url` and Azure send
none on such a turn. Tool-free turns use the turn's effort, else
`OpenAIConfig.reasoning_effort`, else the model default. See
[Reasoning settings on a turn with tools](../guides/ai-thinking.md#reasoning-settings-on-a-turn-with-tools).

The catalogue also says what Chat Completions refuses a model, and the
request fails before it is sent, with a non-retryable `ProviderError`: GPT-6
Astra and GPT-6.1 Sol take no function tools there (the Responses API
serves them with tools), and the `-pro` models are not chat models there. A
custom `base_url` leaves it to the server.

## Per-Room AI Configuration

AI channel settings can be overridden per-room using binding metadata:

```python
# Default AI channel
ai = AIChannel("ai", provider=anthropic, system_prompt="Default assistant")
kit.register_channel(ai)

# Override per room
kit.attach_channel(room_id, "ai", metadata={
    "system_prompt": "You are a customer support agent for Acme Corp.",
    "temperature": 0.3,  # More deterministic
    "max_tokens": 2048,
})
```

### Silent Observer Pattern (Meeting Notes)

```python
# Attach AI as note-taker
kit.attach_channel(meeting_room_id, "ai", metadata={
    "system_prompt": """You are a meeting note-taker.
    Listen to the conversation silently.
    When someone says 'meeting ended', compile and send a summary.""",
})

# Mute so AI listens but doesn't respond
await kit.mute(meeting_room_id, "ai")

# Later, unmute to let AI send summary
await kit.unmute(meeting_room_id, "ai")
```

## Tools/Function Calling

Tools can be passed via binding metadata for function calling:

```python
kit.attach_channel(room_id, "ai", metadata={
    "tools": [
        {
            "name": "search_knowledge_base",
            "description": "Search the company knowledge base",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string"}
                },
                "required": ["query"]
            }
        }
    ]
})
```

Tool calls are returned in `AIResponse.tool_calls`:

```python
response = await provider.generate(context)
for tool_call in response.tool_calls:
    print(f"Tool: {tool_call.name}, Args: {tool_call.arguments}")
```

## Gemini Provider

::: roomkit.providers.gemini.ai.GeminiAIProvider

::: roomkit.providers.gemini.config.GeminiConfig

::: roomkit.providers.gemini.vertex.GeminiVertexProvider

::: roomkit.providers.gemini.vertex.GeminiVertexConfig

### Usage

```python
from roomkit.providers.gemini.ai import GeminiAIProvider
from roomkit.providers.gemini.config import GeminiConfig

config = GeminiConfig(api_key="your-api-key")
provider = GeminiAIProvider(config)

# Use with AIChannel
ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[gemini]`

A tool result whose call was refused, failed, blocked, served by nothing or
cancelled goes to Gemini (and Vertex) under the `error` key of its function
response, any other under `result`; the text the model reads is the same
either way. Gemini Live does not flag it yet.

### Schema Cleaning

Gemini rejects extra JSON Schema fields common in MCP/OpenAPI tool definitions. RoomKit auto-cleans schemas when building `FunctionDeclaration` objects:

::: roomkit.providers.gemini.schema.clean_gemini_schema

## Meta Muse Spark Provider

::: roomkit.providers.meta.ai.MetaAIProvider

::: roomkit.providers.meta.config.MetaConfig

Muse Spark cannot turn reasoning off: `reasoning_effort="none"` is sent as
`"minimal"`. See the [Meta Model API guide](../guides/meta.md).

## vLLM Provider (Local LLM)

::: roomkit.providers.vllm.VLLMConfig

::: roomkit.providers.vllm.create_vllm_provider

### Usage

```python
from roomkit.providers.vllm import create_vllm_provider, VLLMConfig
from roomkit.channels.ai import AIChannel

# Configure connection to a local vLLM server
config = VLLMConfig(
    model="meta-llama/Llama-3.1-8B-Instruct",
    base_url="http://localhost:8000/v1",
)

# Factory returns an OpenAIAIProvider pointed at your vLLM server
provider = create_vllm_provider(config)

# Use with AIChannel like any other AI provider
ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[vllm]`

## llama.cpp Provider (Local LLM, nothing to run beside it)

RoomKit downloads the llama.cpp build for the machine and the model, runs
`llama-server`, and stops it on `close()`. See the
[llama.cpp guide](../guides/llamacpp.md).

::: roomkit.providers.llamacpp.LlamaCppConfig

::: roomkit.providers.llamacpp.LlamaCppAIProvider

### Usage

```python
from roomkit.providers.llamacpp import LlamaCppAIProvider, LlamaCppConfig

provider = LlamaCppAIProvider(
    LlamaCppConfig(model="unsloth/Qwen3-4B-Instruct-2507-GGUF:Q4_K_M")
)
await provider.start()  # optional: load the model before the first request
```

Install with: `pip install roomkit[llamacpp]`

## Anthropic Provider

::: roomkit.providers.anthropic.ai.AnthropicAIProvider

::: roomkit.providers.anthropic.config.AnthropicConfig

A tool result whose call was refused, failed, blocked, served by nothing or
cancelled goes to Anthropic with `is_error` set on its `tool_result` block;
the text the model reads is the same as on any other provider.

### Usage

```python
from roomkit.providers.anthropic.ai import AnthropicAIProvider
from roomkit.providers.anthropic.config import AnthropicConfig
from roomkit.channels.ai import AIChannel

config = AnthropicConfig(api_key="your-api-key", model="claude-opus-5")
provider = AnthropicAIProvider(config)

ai_channel = AIChannel("ai", provider=provider)
```

### Per-request credentials

The configured key remains the default. In a multi-tenant host where a caller
uses their own Anthropic subscription, a `BEFORE_AI_GENERATION` hook can select
that credential for one turn without rebuilding the shared provider:

```python
from roomkit import HookResult, HookTrigger
from roomkit.providers.ai import API_KEY_METADATA_KEY

@kit.hook(HookTrigger.BEFORE_AI_GENERATION)
async def select_anthropic_key(event, ctx):
    key = await tenant_secrets.anthropic_key(ctx.room.metadata["tenant_id"])
    event.ai_context.metadata[API_KEY_METADATA_KEY] = key
    return HookResult.allow()
```

RoomKit stores the value as a Pydantic secret, so rendering or serializing the
context redacts it. An absent, empty, or non-string value — or one equal to the
configured key — falls back to `AnthropicConfig.api_key` and its shared client.

Per-key clients are cached in a pool bounded by a soft limit: only entries whose
last turn has finished are evicted and closed. A burst of distinct credentials
may hold the pool briefly above that limit, and it trims itself as those turns
end, because closing a client underneath an in-flight stream would break a
response that has nothing to do with the new caller.

Install with: `pip install roomkit[anthropic]`

## OpenAI Provider

::: roomkit.providers.openai.ai.OpenAIAIProvider

::: roomkit.providers.openai.config.OpenAIConfig

### Usage

```python
from roomkit.providers.openai.ai import OpenAIAIProvider
from roomkit.providers.openai.config import OpenAIConfig
from roomkit.channels.ai import AIChannel

config = OpenAIConfig(api_key="your-api-key", model="gpt-5.6-sol")
provider = OpenAIAIProvider(config)

ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[openai]`

## Cerebras Provider

::: roomkit.providers.cerebras.ai.CerebrasAIProvider

::: roomkit.providers.cerebras.config.CerebrasConfig

Install with `pip install "roomkit[cerebras]"`. The provider uses the shared
OpenAI-compatible async transport and supports text, streaming, reasoning,
function calls and image input on vision-capable models.

```python
import os

from roomkit import AIChannel, CerebrasAIProvider, CerebrasConfig

provider = CerebrasAIProvider(CerebrasConfig(
    api_key=os.environ["CEREBRAS_API_KEY"],
    model="gpt-oss-120b",
    reasoning_effort="low",
))
ai = AIChannel("assistant", provider=provider)
```

Choose `model` explicitly. `await provider.list_models()` queries the models
available to the account; `CerebrasAIProvider.available_models()` supplies
offline capabilities, context limits and dated prices. The snapshot includes
GPT OSS 120B, Qwen 3.8 27B and Gemma 4 31B; Gemma may require a dedicated
endpoint. Qwen uses a conservative 65,536-token limit covering the trial tier.
Its model card advertises a larger window on paid tiers. Unknown model ids
remain usable, with unknown context size and vision disabled.

`reasoning_effort` remains active when tools are supplied; a per-turn
`AIContext.reasoning_effort` overrides the provider setting. GPT OSS accepts
`low`, `medium` or `high`. Qwen 3.8 also accepts `none` to disable reasoning.
`thinking_budget` is not translated into an effort level. Available values
depend on the selected model, as described in the
[Cerebras reasoning guide](https://inference-docs.cerebras.ai/capabilities/reasoning).

`reasoning_format="parsed"` is the default, keeping thinking separate from
answer text. Historical `AIThinkingPart` values are sent in the assistant's
`reasoning` field, including across tool rounds. `clear_thinking` is optional
and should only be set for a model supporting it. Selecting `raw` can mix
reasoning into visible text; GPT OSS supplies no separator in that mode.

The output cap uses `max_completion_tokens`. Token usage is collected from
Cerebras's final streaming chunk without requesting `stream_options`.
Cache reads are reported separately and priced at the ordinary input rate.
Errors and latency metrics identify the provider as `cerebras`; SDK retries
default to zero so RoomKit's retry policy remains in control.

See `examples/cerebras_ai.py` for a complete conversation.

## Mistral Provider

::: roomkit.providers.mistral.ai.MistralAIProvider

::: roomkit.providers.mistral.config.MistralConfig

### Usage

```python
from roomkit.providers.mistral.ai import MistralAIProvider
from roomkit.providers.mistral.config import MistralConfig
from roomkit.channels.ai import AIChannel

config = MistralConfig(api_key="your-api-key")
provider = MistralAIProvider(config)

ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[mistral]`

## Azure Provider

::: roomkit.providers.azure.ai.AzureAIProvider

::: roomkit.providers.azure.config.AzureAIConfig

### Usage

```python
from roomkit.providers.azure.ai import AzureAIProvider
from roomkit.providers.azure.config import AzureAIConfig
from roomkit.channels.ai import AIChannel

config = AzureAIConfig(
    azure_endpoint="https://your-resource.openai.azure.com/",
    api_key="your-api-key",
    deployment="your-deployment-name",
)
provider = AzureAIProvider(config)

ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[azure]`

## Ollama Provider (Local / Cloud LLM)

Native provider for [Ollama](https://ollama.com). Calls `/api/chat` directly, so
the `think` parameter and streamed reasoning work without `<think>` tag parsing.
See [AI Thinking — Native Ollama provider](../guides/ai-thinking.md#native-ollama-provider-authentication)
for thinking, authentication, and sampling options (`temperature`, `num_ctx`,
`top_p`, `top_k`, `min_p`, `keep_alive`).

::: roomkit.providers.ollama.ai.OllamaAIProvider

::: roomkit.providers.ollama.config.OllamaConfig

### Usage

```python
from roomkit.providers.ollama import OllamaAIProvider, OllamaConfig
from roomkit.channels.ai import AIChannel

config = OllamaConfig(host="http://localhost:11434", model="llama3.2")
provider = OllamaAIProvider(config)

ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[ollama]`

A tool's nested parameters reach the model: a nested object's `properties`
and `required`, and an `anyOf`. ollama-python drops them from a declaration
before the request leaves (ollama/ollama-python#724; measured on 0.6.2 and
0.6.3), although the Ollama server reads them. Until a release sends them, a
request that declares tools goes through the SDK's own request method with the
declarations as given (`roomkit.providers.ollama.sdk_patch`). The server
itself drops a `$ref` and the constraints (`minLength` and the like), so a
schema meant for Ollama spells those out in its descriptions.

## Streaming

::: roomkit.providers.ai.base.StreamEvent

::: roomkit.providers.ai.base.StreamTextDelta

::: roomkit.providers.ai.base.StreamThinkingDelta

::: roomkit.providers.ai.base.StreamToolCall

::: roomkit.providers.ai.base.StreamDone

## Response Parts

::: roomkit.providers.ai.base.AIThinkingPart

::: roomkit.providers.ai.base.AIToolCallPart

::: roomkit.providers.ai.base.AIToolResultPart

::: roomkit.providers.ai.base.ProviderError
