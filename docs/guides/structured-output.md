# Structured Output (Response Schema)

Some calls do not want prose back. A triage step wants a department and an
urgency flag, a judge wants a verdict, a classifier wants a label. Asking for
JSON in the prompt and parsing whatever comes back works most of the time, and
fails on the answer that wraps the object in a sentence or stops halfway
through it.

`AIContext.response_schema` removes that guesswork. It carries a JSON Schema,
the provider constrains its output natively, and `generate()` returns one JSON
document that satisfies the schema. If it cannot, it raises an error that says
why. It never returns prose in its place.

## Quick start

```python
import json

from roomkit import ResponseSchemaError
from roomkit.providers.ai import AIContext, AIMessage
from roomkit.providers.gemini import GeminiAIProvider, GeminiConfig

TRIAGE = {
    "type": "object",
    "properties": {
        "department": {"type": "string", "enum": ["billing", "technical", "sales", "other"]},
        "urgent": {"type": "boolean"},
        "summary": {"type": "string", "description": "One short sentence."},
    },
    "required": ["department", "urgent", "summary"],
    "additionalProperties": False,
}

provider = GeminiAIProvider(GeminiConfig(api_key="..."))
context = AIContext(
    system_prompt="You triage customer support messages.",
    messages=[AIMessage(role="user", content="I was charged twice for March.")],
    response_schema=TRIAGE,
)

try:
    response = await provider.generate(context)
except ResponseSchemaError as exc:
    print(f"no triage: {exc.reason}")
else:
    triage = json.loads(response.content)
```

`response.content` stays a string. Turning it into a typed value (with
`json.loads`, or a Pydantic model's `model_validate_json`) is the caller's work.

A runnable version is in `examples/ai_response_schema.py`. It runs offline on
the mock, or against Gemini, Anthropic, OpenAI or a local Ollama.

## The portable subset

Every provider reads its own dialect of JSON Schema. OpenAI's strict mode
wants every property required; Anthropic refuses numeric bounds; Gemini reads
its own subset. RoomKit accepts only what they all take, so a schema written
once runs on any provider that supports response schemas:

| Keyword | Rule |
|---|---|
| `type` | `object`, `array`, `string`, `number`, `integer` or `boolean`, as a single string. The root is an `object`. |
| `properties` | Required on an object, possibly empty. |
| `required` | Required on an object, and lists every property, each once. |
| `additionalProperties` | Required on an object, and `false`. |
| `items` | Required on an array. |
| `enum` | On a string only: a non-empty list of distinct strings. |
| `title`, `description` | Allowed anywhere, as strings. The model reads them. |

Nothing else passes: no `null` or optional fields, no `anyOf`, no `$ref`, no
`minimum` or `maxLength`. For an answer that may be absent, use a required
field with an agreed empty value (`""`, `[]`).

The check runs when the context is built, when a hook assigns the field, and
through `model_copy(update=...)`. A schema outside the subset fails there,
before any provider is called. `check_portable_schema(schema)` runs the same
check on its own, for a schema you build at startup.

## Providers

| Provider | Native field | Supported |
|---|---|---|
| OpenAI, Azure, xAI, Cerebras, OpenRouter, LiteLLM, vLLM, llama.cpp | strict `json_schema` `response_format` | yes |
| DeepSeek, Qwen | — | no: only free-form JSON mode is documented |
| Anthropic | `output_config.format` | yes |
| Gemini, Gemini Vertex | `response_mime_type` + `response_json_schema` | yes |
| Mistral | strict `json_schema` `response_format` | yes |
| Ollama | `format` | yes |
| PolarGrid | `json_schema` `response_format` | yes |

`provider.supports_response_schema` gives the provider's answer. It is a
default, not a promise for every model: a model older than the feature may
still reject the request, which surfaces as a `ProviderError`.

Whether an OpenAI-compatible server applies the format depends on the server,
so `OpenAIConfig`, `AzureAIConfig` and `VLLMConfig` take
`supports_response_schema=` to state it:

```python
OpenAIConfig(
    api_key="...",
    base_url="http://my-server:8000/v1",
    model="my-model",
    supports_response_schema=False,  # this server ignores response_format
)
```

## When there is no document

`ResponseSchemaError` is a `ProviderError` that is never retryable: the same
request fails the same way. Its `reason` tells the cases apart:

| `reason` | Meaning |
|---|---|
| `unsupported` | The call cannot carry a schema: the provider does not support one, the turn also has `tools`, or a streaming method received it. Raised before any request is sent. |
| `refusal` | The model declined to answer: a refusal field (OpenAI), a `refusal` stop reason (Anthropic), a safety stop (Gemini), a content filter. |
| `truncated` | The output cap cut the document. Raise `max_tokens`; a reasoning model spends part of it thinking. |
| `invalid_json` | The text is not JSON. That happens with a server that accepted the constraint and did not apply it. |

## Running on any provider

A component that must work with every provider checks the property first, and
where it is false asks for JSON in the prompt and parses the answer itself:

```python
if provider.supports_response_schema:
    context = context.model_copy(update={"response_schema": TRIAGE})
else:
    context.system_prompt += "\nAnswer with one JSON object: " + json.dumps(TRIAGE)
```

## Not in this version

- **Streaming.** `generate_stream()` and `generate_structured_stream()` refuse
  a schema. What a half-written document means mid-stream is not defined yet.
- **Tools.** A turn cannot carry both `tools` and a schema, so an `AIChannel`
  turn with tools cannot use one either.

The rules live in [RFC §6.7](https://github.com/roomkit-live/roomkit-specs).

## Testing

`MockAIProvider(response_schema=True)` follows the same contract. The scripted
answer must be JSON, and an `AIResponse` scripted with
`finish_reason="refusal"` or `"length"` raises the matching error:

```python
from roomkit.providers.ai import AIResponse, MockAIProvider

provider = MockAIProvider(
    ai_responses=[AIResponse(content="", finish_reason="refusal")],
    response_schema=True,
)
```
