# Gemini on Vertex AI

`GeminiVertexProvider` runs Google's Gemini models through **Vertex AI** in a pinned Google Cloud region instead of the public Gemini Developer API. Same models, same code — but the request is processed in the region you choose and is **not retained to train Google's models**. That makes it the backend to reach for when data residency matters (e.g. Québec Law 25 / PIPEDA, EU data boundaries).

It is a thin subclass of `GeminiAIProvider` — only the client construction differs (Vertex mode + Application Default Credentials instead of an API key). Generation, streaming, thinking, and the model catalog are all inherited.

## Install & authenticate

```bash
pip install roomkit[gemini]              # no extra dependency — same SDK
gcloud auth application-default login    # provides ADC
```

Vertex uses **Application Default Credentials** — the standard Google Cloud chain (`gcloud auth application-default login`, `GOOGLE_APPLICATION_CREDENTIALS`, or a workload-identity service account). There is no API key.

## Quick start

```python
from roomkit import AIChannel, RoomKit
from roomkit.providers.gemini import GeminiVertexProvider, GeminiVertexConfig

provider = GeminiVertexProvider(
    GeminiVertexConfig(
        project="my-gcp-project",
        location="northamerica-northeast1",   # Montréal — required, no default
        model="gemini-3.1-flash-lite",
    )
)

kit = RoomKit()
ai = AIChannel("assistant", provider=provider, system_prompt="You are helpful.")
kit.register_channel(ai)
```

## Why `location` is required

`location` has **no default on purpose**. Data residency is the reason to use Vertex, and a convenience default (e.g. `global`) could route requests out of your region and quietly defeat it. Pin it explicitly to the region your compliance regime requires:

| Region | Location id |
|--------|-------------|
| Montréal | `northamerica-northeast1` |
| Toronto | `northamerica-northeast2` |
| Belgium (EU) | `europe-west1` |
| Iowa (US) | `us-central1` |

See [Vertex AI locations](https://cloud.google.com/vertex-ai/docs/general/locations) for the full list and which models each region serves.

## How it works

| Aspect | Behaviour |
|--------|-----------|
| Client | `genai.Client(vertexai=True, project=…, location=…)` — the same `google-genai` SDK as `GeminiAIProvider`, in Vertex mode |
| Auth | Application Default Credentials (no API key); `GeminiVertexConfig.api_key` is optional and ignored |
| Config | `GeminiVertexConfig` **subclasses** `GeminiConfig`, inheriting every generation field (`model`, `max_tokens`, `temperature`, `thinking_level`) so the two never drift |
| Models | The same Gemini catalog — `available_models()` / `list_models()` inherited |
| Thinking | Inherited: `thinking_level` requests thought summaries, surfaced as `StreamThinkingDelta` |

See `examples/gemini_vertex_ai.py` for an end-to-end run.
