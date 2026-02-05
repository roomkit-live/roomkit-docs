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
from roomkit import GeminiAIProvider, GeminiConfig

config = GeminiConfig(api_key="your-api-key")
provider = GeminiAIProvider(config)

# Use with AIChannel
ai_channel = AIChannel("ai", provider=provider)
```

Install with: `pip install roomkit[gemini]`

## vLLM Provider (Local LLM)

::: roomkit.providers.vllm.VLLMConfig

::: roomkit.providers.vllm.create_vllm_provider

### Usage

```python
from roomkit import create_vllm_provider, VLLMConfig
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
