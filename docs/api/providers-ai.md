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

::: roomkit.providers.ai.mock.MockAIProvider

## Modern model request defaults

`OpenAIConfig()` profiles first-party `gpt-5`/o-series models to use
`max_completion_tokens` and omit a custom temperature. `AnthropicConfig()`
profiles modern Claude reasoning models to use adaptive thinking and omit
temperature. These defaults prevent the libraries' current flagship model ids
from being paired with parameters those models reject.

Explicit compatibility flags always win. A custom `base_url` also keeps the
conservative legacy defaults, since OpenAI-compatible and Anthropic-compatible
proxies may implement the older request shape.

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

### Schema Cleaning

Gemini rejects extra JSON Schema fields common in MCP/OpenAPI tool definitions. RoomKit auto-cleans schemas when building `FunctionDeclaration` objects:

::: roomkit.providers.gemini.schema.clean_gemini_schema

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

## Anthropic Provider

::: roomkit.providers.anthropic.ai.AnthropicAIProvider

::: roomkit.providers.anthropic.config.AnthropicConfig

### Usage

```python
from roomkit.providers.anthropic.ai import AnthropicAIProvider
from roomkit.providers.anthropic.config import AnthropicConfig
from roomkit.channels.ai import AIChannel

config = AnthropicConfig(api_key="your-api-key")
provider = AnthropicAIProvider(config)

ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[anthropic]`

## OpenAI Provider

::: roomkit.providers.openai.ai.OpenAIAIProvider

::: roomkit.providers.openai.config.OpenAIConfig

### Usage

```python
from roomkit.providers.openai.ai import OpenAIAIProvider
from roomkit.providers.openai.config import OpenAIConfig
from roomkit.channels.ai import AIChannel

config = OpenAIConfig(api_key="your-api-key")
provider = OpenAIAIProvider(config)

ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[openai]`

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
