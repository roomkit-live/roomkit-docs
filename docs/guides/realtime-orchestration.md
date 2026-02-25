# Speech-to-Speech Orchestration

Multi-agent orchestration using Gemini Live (or OpenAI Realtime) speech-to-speech, where the AI provider handles audio recognition, reasoning, and speech generation natively — no STT/LLM/TTS pipeline.

## Overview

Traditional orchestration routes text events between `Agent` channels backed by an `AIProvider`. Each agent is an intelligence channel in the room, and `VoiceChannel` handles the audio conversion (STT on the way in, TTS on the way out).

Speech-to-speech orchestration uses the same `Agent` definitions — same `role`, `description`, `scope`, `voice`, `system_prompt` — but wires them into a `RealtimeVoiceChannel` instead. The AI reasoning happens inside the provider (Gemini Live), not in a separate channel. On handoff, the framework reconnects the provider session with the new agent's configuration, preserving conversation history via Gemini's session resumption.

```
Traditional (STT → LLM → TTS):
  User speaks → VoiceChannel (STT) → text → Agent (AIChannel) → text → VoiceChannel (TTS)

Speech-to-speech:
  User speaks → RealtimeVoiceChannel → Gemini Live (audio in → AI → audio out) → User hears
                                         ↑
                                    Agent config (system_prompt, voice, tools)
                                    swapped on handoff via session resumption
```

The developer writes nearly identical code for both modes. The only difference is whether `Agent` has a `provider` or not:

```python
# Traditional (STT → LLM → TTS)
triage = Agent(
    "agent-triage",
    provider=GeminiAIProvider(config),    # <-- has AI provider
    role="Triage receptionist",
    system_prompt="Greet callers warmly. Under 30 words.",
    voice="en-US-Journey-F",             # TTS voice ID
)
voice = VoiceChannel("voice", stt=deepgram, tts=elevenlabs)

# Speech-to-speech
triage = Agent(
    "agent-triage",                       # <-- no provider (config-only)
    role="Triage receptionist",
    system_prompt="Greet callers warmly. Under 30 words.",
    voice="Aoede",                        # Gemini native voice
)
voice = RealtimeVoiceChannel("voice", provider=GeminiLiveProvider(...), transport=...)
```

Both use the same `pipeline.install()` call. The framework detects the channel type and wires things differently under the hood.

## How it works

### Handoff via session resumption

Gemini Live does not support mid-session configuration changes. However, it supports **session resumption with different configuration** — you can disconnect and reconnect with a new `system_instruction`, `voice`, and `tools` while preserving the conversation history. The model parameter is the only thing that cannot change.

On handoff, the framework:

1. Builds the new agent's config (system_prompt + identity block, voice, handoff tools)
2. Disconnects the current Gemini session
3. Reconnects with the new config + the session resumption handle
4. Gemini remembers the full conversation but responds with the new agent's personality/voice

The reconnect takes ~200-500ms — a natural pause, similar to a phone transfer.

### Wiring

When `pipeline.install()` detects that `voice_channel_id` points to a `RealtimeVoiceChannel`:

1. **Agents are NOT registered as channels** — they have no provider, they're config-only
2. **The active agent's config is applied to the RealtimeVoiceChannel** — system_prompt, voice, and tools are passed to `provider.connect()`
3. **The handoff tool is registered on the RealtimeVoiceChannel** — Gemini calls it natively via its built-in tool calling
4. **On handoff, `reconfigure_session()` is called** — disconnects and reconnects with the new agent's config

```
pipeline.install(kit, [triage, advisor], voice_channel_id="voice")

Internally:
  1. Does NOT call kit.register_channel() for agents
  2. Builds per-agent handoff tools (enum-constrained targets)
  3. Sets initial agent config on RealtimeVoiceChannel
  4. Registers handoff tool_handler on RealtimeVoiceChannel
  5. HandoffHandler.handle() → rtv.reconfigure_session(session, new_config)
```

### Agent identity in speech-to-speech

Agent identity (`role`, `description`, `scope`) is appended to the `system_prompt` passed to `provider.connect()`, just as it would be appended to the AIChannel's prompt in traditional mode. The AI sees the same identity block regardless of mode:

```
Greet callers warmly. Under 30 words.

--- Agent Identity ---
Role: Triage receptionist
Description: Routes callers to the right financial specialist
Scope: Financial advisory services only
```

## Quick start

```python
from roomkit import (
    Agent,
    ConversationPipeline,
    PipelineStage,
    RealtimeVoiceChannel,
    RoomKit,
)
from roomkit.providers.gemini.realtime import GeminiLiveProvider
from roomkit.voice.backends.local import LocalAudioBackend

# 1. Define agents (no provider — config-only for speech-to-speech)
triage = Agent(
    "agent-triage",
    role="Triage receptionist",
    description="Routes callers to the right specialist",
    system_prompt="Greet callers. Ask what they need. Under 30 words.",
    voice="Aoede",
)

advisor = Agent(
    "agent-advisor",
    role="Portfolio advisor",
    description="Provides financial and investment advice",
    system_prompt="Give concise financial advice. Under 50 words.",
    voice="Charon",
)

# 2. Define pipeline (same as traditional orchestration)
pipeline = ConversationPipeline(
    stages=[
        PipelineStage(phase="triage", agent_id="agent-triage", next="advising"),
        PipelineStage(phase="advising", agent_id="agent-advisor", next=None),
    ],
)

# 3. Create RealtimeVoiceChannel with Gemini Live provider
kit = RoomKit()

provider = GeminiLiveProvider(api_key="...")
transport = LocalAudioBackend(
    input_sample_rate=24000,
    output_sample_rate=24000,
)

voice = RealtimeVoiceChannel(
    "voice",
    provider=provider,
    transport=transport,
)
kit.register_channel(voice)

# 4. One-liner: wires everything (detects RealtimeVoiceChannel automatically)
router, handler = pipeline.install(kit, [triage, advisor], voice_channel_id="voice")

# 5. Create room, start session — same as any realtime voice app
await kit.create_room(room_id="call-1")
await kit.attach_channel("call-1", "voice")
session = await voice.start_session("call-1", "caller-1", connection=None)
```

When the caller asks for investment advice, Gemini (as the triage agent) calls the `handoff_conversation` tool. The framework reconnects the Gemini session with the advisor's config — the caller hears the advisor's voice and personality without any gap in context.

## Gemini Live constraints

| Aspect | Supported | Notes |
|--------|-----------|-------|
| Change system_instruction on reconnect | Yes | Via session resumption |
| Change voice on reconnect | Yes | Via session resumption |
| Change tools on reconnect | Yes | Via session resumption |
| Change model on reconnect | No | Must be same model |
| Mid-session config update (no reconnect) | No | No `session.update` API |
| Conversation history preserved | Yes | Session resumption handle |
| Reconnect latency | ~200-500ms | Natural handoff pause |

## Comparison with traditional orchestration

| Aspect | Traditional (STT→LLM→TTS) | Speech-to-Speech |
|--------|---------------------------|------------------|
| Agent definition | `Agent(..., provider=AIProvider)` | `Agent(...)` (no provider) |
| Voice channel | `VoiceChannel` with STT + TTS | `RealtimeVoiceChannel` |
| AI reasoning | In Agent (AIChannel) | In provider (Gemini Live) |
| Latency | Higher (STT + LLM + TTS) | Lower (native audio) |
| Voice quality | Depends on TTS provider | Native (Gemini voices) |
| Provider flexibility | Mix providers per agent | Single provider, all agents |
| Tool calling | Via AIChannel tool loop | Via provider native tools |
| Handoff mechanism | Route events to different Agent | Reconnect provider session |
| `install()` call | Identical | Identical |

## Implementation details

### `reconfigure_session()` on RealtimeVoiceChannel

```python
async def reconfigure_session(
    self,
    session: VoiceSession,
    *,
    system_prompt: str | None = None,
    voice: str | None = None,
    tools: list[dict[str, Any]] | None = None,
) -> None:
    """Switch agent config by reconnecting with session resumption.

    Disconnects the current provider session and reconnects with
    new configuration. Gemini preserves conversation history via
    the session resumption handle.
    """
```

### `reconfigure()` on RealtimeVoiceProvider

```python
async def reconfigure(
    self,
    session: VoiceSession,
    *,
    system_prompt: str | None = None,
    voice: str | None = None,
    tools: list[dict[str, Any]] | None = None,
    temperature: float | None = None,
    provider_config: dict[str, Any] | None = None,
) -> None:
    """Rebuild config and reconnect with session resumption."""
```

GeminiLiveProvider implements this by:

1. Building a new `LiveConnectConfig` from the provided parameters
2. Storing it as `_live_configs[session.id]`
3. Calling `_reconnect(session)` which uses the stored resumption handle

### Config-only Agent

When `Agent` is created without a `provider`, it uses an internal `_NullAIProvider` placeholder. The agent is still an `AIChannel` instance (`isinstance` checks work), but calling `generate()` raises a clear error. This is intentional — config-only agents are never registered as channels.

```python
# Config-only (speech-to-speech)
triage = Agent("agent-triage", role="Triage", system_prompt="...")
assert isinstance(triage, AIChannel)  # True
# triage._provider.generate() would raise RuntimeError

# Full agent (traditional)
triage = Agent("agent-triage", provider=GeminiAIProvider(config), role="Triage", ...)
# triage._provider.generate() works normally
```

### `install()` detection logic

```python
def install(self, kit, agents, *, voice_channel_id=None, ...):
    vc = kit._channels.get(voice_channel_id) if voice_channel_id else None

    if isinstance(vc, RealtimeVoiceChannel):
        # Speech-to-speech mode
        self._wire_realtime(kit, agents, vc)
    else:
        # Traditional mode (existing behavior)
        self._wire_handoff(agents, handler)
        if voice_channel_id:
            self._wire_voice_map(kit, agents, voice_channel_id)
```

In speech-to-speech mode, `_wire_realtime()`:

1. Builds the initial agent's config with identity block
2. Applies it to the RealtimeVoiceChannel (system_prompt, voice, tools)
3. Builds per-agent handoff tools (enum-constrained targets with descriptions)
4. Registers a `tool_handler` on the RealtimeVoiceChannel that intercepts `handoff_conversation` calls
5. The tool handler calls `HandoffHandler.handle()` which triggers `reconfigure_session()`

## Examples

- [`examples/orchestration_realtime_triage.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/orchestration_realtime_triage.py) — Voice triage with Gemini Live speech-to-speech
