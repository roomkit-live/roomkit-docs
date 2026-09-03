# Gemini on Vertex AI

`GeminiVertexProvider` runs Google's Gemini models through **Vertex AI** in a pinned Google Cloud region instead of the public Gemini Developer API. Same models, same code — but the request is processed in the region you choose and is **not retained to train Google's models**. That makes it the backend to reach for when data residency matters (e.g. Québec Law 25 / PIPEDA, EU data boundaries).

It is a thin subclass of `GeminiAIProvider` — only the client construction (Vertex mode, and an identity that is not an API key) and the [billing labels](#billing-labels) on each request differ. Generation, streaming, thinking, and the model catalog are all inherited.

## Install & authenticate

```bash
pip install roomkit[gemini]              # no extra dependency — same SDK
gcloud auth application-default login    # the ambient identity, for local dev
```

There is no API key on Vertex. The caller is authenticated by one of three identities, read in this order:

| Config | Identity | Reach for it when |
|--------|----------|-------------------|
| `impersonate_service_account` | a service account you **borrow**, through short-lived tokens | the project's owner cannot, or will not, hand out a key |
| `service_account_json` | a service account whose **key you hold** | one deployment serves several projects |
| *(neither set)* | Application Default Credentials — whoever the process is | you own the project you are calling |

The two fields combine rather than exclude each other: when both are set, the key is the identity that borrows.

### Application Default Credentials (the default)

The standard Google Cloud chain — `gcloud auth application-default login`, `GOOGLE_APPLICATION_CREDENTIALS`, or workload identity. Set neither field and nothing changes from earlier releases:

```python
GeminiVertexConfig(project="my-gcp-project", location="northamerica-northeast1")
```

ADC answers *"who is this machine"*. That is the right question for a deployment that owns the project it calls, and the wrong one everywhere else — see below.

### A service-account key

```python
GeminiVertexConfig(
    project="client-42",
    location="northamerica-northeast1",
    service_account_json=os.environ["CLIENT_42_SA_KEY"],   # the key file's contents, as JSON
)
```

Pass the JSON **contents**, not a path. The identity then travels with the configuration instead of with the process, which is what lets one server serve one project per tenant: the ambient identity belongs to whoever runs the server, so a caller naming someone else's project gets `PERMISSION_DENIED` no matter what it puts in `project`.

### Borrowing a service account

Recent Google Cloud organizations enforce `constraints/iam.disableServiceAccountKeyCreation` by default, so a project owner often *cannot* give you a key even when willing. Impersonation replaces the secret with a grant:

1. The owner grants your deployment's own identity `roles/iam.serviceAccountTokenCreator` on one of their service accounts.
2. RoomKit calls Vertex as that account, with short-lived tokens nobody downloads.
3. The owner revokes the grant from their side, in one command, without telling you.

```python
GeminiVertexConfig(
    project="client-42",
    location="northamerica-northeast1",
    impersonate_service_account="roomkit@client-42.iam.gserviceaccount.com",
)
```

The borrowing identity is `service_account_json` when one is configured, otherwise ADC — so the deployment still needs credentials of its own. Without any, the call fails with `Cannot borrow …: this deployment has no Google credentials of its own to borrow it with.`

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

## Billing labels

One Google Cloud project often serves several tenants or partners, and Cloud Billing reports the project's Vertex spend as one number. `labels` splits it: the dict rides every request, and the billing report groups the charges by label key and value.

```python
GeminiVertexConfig(
    project="my-gcp-project",
    location="northamerica-northeast1",
    labels={"tenant": "acme", "partner": "north-reseller"},
)
```

What to expect from it:

- **Metadata only.** A label sets no quota and no limit, and it changes nothing in the answer.
- **Late.** Labelled charges reach the billing report 24 to 48 hours after the request. Meter what a tenant consumed from the usage each response carries; the report is for attribution, never a source of truth.
- **Constant per provider.** A provider serves one channel, a channel one agent, an agent one tenant, so the label belongs on the configuration, not on the turn.
- **Validated at configuration.** Google's rules are enforced when `GeminiVertexConfig` is built, so a bad label fails at startup rather than on a tenant's first message: at most 64 labels; keys 1 to 63 characters starting with a lowercase letter, values 0 to 63; lowercase letters, digits, `_` and `-` only (international characters allowed).
- **Vertex only.** The Gemini Developer API refuses the field, which is why it lives on `GeminiVertexConfig` and not on `GeminiConfig`.

## How it works

| Aspect | Behaviour |
|--------|-----------|
| Client | `genai.Client(vertexai=True, project=…, location=…)` — the same `google-genai` SDK as `GeminiAIProvider`, in Vertex mode |
| Auth | No API key. `impersonate_service_account`, else `service_account_json`, else Application Default Credentials; `GeminiVertexConfig.api_key` is optional and ignored |
| Config | `GeminiVertexConfig` **subclasses** `GeminiConfig`, inheriting every generation field (`model`, `max_tokens`, `temperature`, `thinking_level`) so the two never drift |
| Models | The same Gemini catalog — `available_models()` / `list_models()` inherited |
| Thinking | Inherited: `thinking_level` requests thought summaries, surfaced as `StreamThinkingDelta` |
| Labels | `labels` rides every request as `GenerateContentConfig.labels`; Cloud Billing groups the charges by them. Vertex only: the Developer API refuses the field |

See `examples/gemini_vertex_ai.py` for an end-to-end run.
