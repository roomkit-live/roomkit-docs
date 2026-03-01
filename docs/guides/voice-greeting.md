# Voice Greeting

Greet callers automatically when the voice audio path is ready. RoomKit provides several patterns ranging from zero-code static greetings to fully dynamic LLM-generated responses.

---

## When to use each pattern

| Pattern | Latency | Flexibility | Best for |
|---|---|---|---|
| **1. Agent auto_greet** | Lowest | Static text | Most use cases — zero code |
| **2. Explicit hook + send_greeting()** | Low | Static text, custom timing | Conditional greetings, A/B testing |
| **3. Manual say()** | Low | Static text | Non-agent greetings, pre-rendered audio |
| **4. LLM-generated** | Higher (LLM round-trip) | Dynamic, context-aware | Personalized greetings based on caller data |

---

## Pattern 1: Agent auto_greet (recommended)

Set `greeting` on the Agent and it speaks automatically when the voice session is ready. No hooks or extra code needed.

```python
from roomkit import Agent, RoomKit, VoiceChannel

kit = RoomKit(stt=stt, tts=tts, voice=backend)

voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend)
agent = Agent(
    "agent",
    provider=ai_provider,
    greeting="Welcome to Acme Support! How can I help you today?",
    # auto_greet=True is the default
)
kit.register_channel(voice)
kit.register_channel(agent)
```

The greeting text is sent directly through TTS — no LLM round-trip, so the caller hears it with near-zero latency.

---

## Pattern 2: Explicit hook + send_greeting()

Register an `ON_VOICE_SESSION_READY` hook and call `send_greeting()` yourself. This gives you control over timing and lets you add conditional logic.

```python
from roomkit import Agent, HookExecution, HookTrigger, RoomKit, VoiceChannel
from roomkit.voice.events import VoiceSessionReadyEvent

agent = Agent(
    "agent",
    provider=ai_provider,
    greeting="Welcome to Acme Support!",
    auto_greet=False,  # We'll trigger it ourselves
)
kit.register_channel(voice)
kit.register_channel(agent)

room = await kit.create_room()
await kit.attach_channel(room.id, "voice")
await kit.attach_channel(room.id, "agent")

@kit.hook(HookTrigger.ON_VOICE_SESSION_READY, HookExecution.ASYNC)
async def on_ready(event: VoiceSessionReadyEvent, context: object) -> None:
    # Add any conditional logic here
    await kit.send_greeting(room.id)
```

!!! tip
    Use this pattern when you need to check caller metadata, time of day, or other conditions before deciding which greeting to send.

---

## Pattern 3: Manual say()

Use the hook to call `voice.say()` directly. This bypasses the Agent entirely — useful for non-agent greetings or pre-rendered audio.

```python
from roomkit import HookExecution, HookTrigger, VoiceChannel
from roomkit.voice.events import VoiceSessionReadyEvent

voice = VoiceChannel("voice", stt=stt, tts=tts, backend=backend)
kit.register_channel(voice)

@kit.hook(HookTrigger.ON_VOICE_SESSION_READY, HookExecution.ASYNC)
async def on_ready(event: VoiceSessionReadyEvent, context: object) -> None:
    await voice.say(event.session, "Please hold while we connect you.")
```

---

## Pattern 4: LLM-generated greeting

Inject a synthetic user message via `process_inbound()` to trigger an LLM round-trip. The AI generates a contextual greeting instead of a static string.

```python
from roomkit import Agent, HookExecution, HookTrigger, RoomKit, VoiceChannel
from roomkit.models.delivery import InboundMessage
from roomkit.models.event import TextContent
from roomkit.voice.events import VoiceSessionReadyEvent

agent = Agent(
    "agent",
    provider=ai_provider,
    system_prompt="You are a helpful assistant. When you receive '[session started]', "
                  "greet the caller warmly and ask how you can help.",
    auto_greet=False,  # No static greeting — let the LLM generate one
)
kit.register_channel(voice)
kit.register_channel(agent)

room = await kit.create_room()
await kit.attach_channel(room.id, "voice")
await kit.attach_channel(room.id, "agent")

@kit.hook(HookTrigger.ON_VOICE_SESSION_READY, HookExecution.ASYNC)
async def on_ready(event: VoiceSessionReadyEvent, context: object) -> None:
    inbound = InboundMessage(
        channel_id="voice",
        sender_id=event.session.participant_id,
        content=TextContent(body="[session started]"),
        metadata={"voice_session_id": event.session.id, "source": "greeting"},
    )
    await kit.process_inbound(inbound, room_id=room.id)
```

The synthetic `[session started]` message flows through the full inbound pipeline: the AI channel sees it, generates a response, and the response is broadcast back through the voice channel as speech.

!!! note
    This pattern adds LLM latency before the caller hears the greeting. Use it when you need personalized greetings based on caller context, time of day, or conversation history. For static greetings, prefer patterns 1–3.

---

## Full example

See [`examples/voice_greeting.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/voice_greeting.py) for a runnable example demonstrating all four patterns with mock providers.
