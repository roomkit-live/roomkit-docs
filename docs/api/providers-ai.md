# AI Providers

::: roomkit.AIProvider

::: roomkit.AIContext

::: roomkit.AIMessage

::: roomkit.AITextPart

::: roomkit.AIImagePart

::: roomkit.AITool

::: roomkit.AIToolCall

::: roomkit.AIResponse

::: roomkit.MockAIProvider

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

::: roomkit.GeminiAIProvider

::: roomkit.GeminiConfig

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

::: roomkit.AnthropicAIProvider

::: roomkit.AnthropicConfig

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

::: roomkit.OpenAIAIProvider

::: roomkit.OpenAIConfig

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

::: roomkit.MistralAIProvider

::: roomkit.MistralConfig

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

::: roomkit.AzureAIProvider

::: roomkit.AzureAIConfig

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

## Streaming

::: roomkit.StreamEvent

::: roomkit.StreamTextDelta

::: roomkit.StreamThinkingDelta

::: roomkit.StreamToolCall

::: roomkit.StreamDone

## Response Parts

::: roomkit.AIThinkingPart

::: roomkit.AIToolCallPart

::: roomkit.AIToolResultPart

::: roomkit.ProviderError
