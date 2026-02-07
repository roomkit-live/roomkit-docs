# Technical Documentation

## Technology Stack

| Category | Technology | Version |
|---|---|---|
| **Language** | Python | >= 3.12 |
| **Data Validation** | Pydantic | >= 2.9 |
| **HTTP Client** | httpx | >= 0.27 (optional) |
| **AI - Anthropic** | anthropic SDK | >= 0.30 (optional) |
| **AI - OpenAI** | openai SDK | >= 1.30 (optional) |
| **AI - Gemini** | google-genai | >= 1.0.0 (optional) |
| **AI - Framework** | pydantic-ai | >= 0.1 (optional) |
| **SMS - Twilio** | twilio | >= 9.0 (optional) |
| **Phone Validation** | phonenumbers | >= 8.13 (optional) |
| **Crypto (Telnyx)** | pynacl | >= 1.5 (optional) |
| **Voice - FastRTC** | fastrtc + numpy | latest (optional) |
| **Voice - STT** | Deepgram (via httpx + websockets) | >= 0.27 / >= 13.0 (optional) |
| **Voice - TTS** | ElevenLabs (via httpx + websockets) | >= 0.27 / >= 13.0 (optional) |
| **Build System** | Hatchling | latest |
| **Package Manager** | uv | latest |
| **Test Framework** | pytest | >= 8.0 |
| **Async Testing** | pytest-asyncio | >= 0.24 |
| **Coverage** | pytest-cov | >= 5.0 |
| **Type Checker** | mypy | >= 1.11 (strict mode) |
| **Linter/Formatter** | Ruff | >= 0.6 |
| **Git Hooks** | pre-commit | >= 3.8 |
| **Documentation** | MkDocs + Material | >= 1.6 / >= 9.5 |
| **API Docs** | mkdocstrings[python] | >= 0.27 |
| **CI/CD** | GitHub Actions | -- |

---

## Project Structure

```
roomkit/
├── .github/
│   └── workflows/
│       └── ci.yml                 # GitHub Actions CI pipeline
├── docs/                          # MkDocs documentation source
│   ├── api/                       # Auto-generated API reference
│   ├── index.md                   # Documentation home
│   ├── architecture.md            # Architecture overview
│   ├── technical.md               # This file
│   ├── features.md                # Feature documentation
│   ├── ai-integration.md          # AI integration guide
│   ├── mcp.md                     # MCP integration
│   ├── roomkit-rfc.md             # RFC v11 (design specification)
│   └── cpaas-comparison.md        # CPaaS comparison (Twilio vs RoomKit)
├── examples/                      # Executable example scripts
│   ├── quickstart.py              # WebSocket + AI quickstart
│   ├── anthropic_ai.py            # Anthropic Claude integration
│   ├── openai_ai.py               # OpenAI integration
│   ├── http_webhook.py            # Generic HTTP webhook
│   ├── voicemeup_sms.py           # VoiceMeUp SMS provider
│   ├── elasticemail.py            # ElasticEmail provider
│   └── facebook_messenger.py      # Facebook Messenger integration
├── src/roomkit/                   # Main library source
│   ├── __init__.py                # Public API (~150 exports via __all__)
│   ├── _version.py                # Version string (0.1.1)
│   ├── py.typed                   # PEP 561 type marker
│   ├── ai_docs.py                 # AI documentation helpers
│   ├── core/                      # Framework internals
│   │   ├── framework.py           # RoomKit orchestrator (mixin composition)
│   │   ├── _inbound.py            # InboundMixin (message pipeline)
│   │   ├── _channel_ops.py        # ChannelOpsMixin (channel management)
│   │   ├── _room_lifecycle.py     # RoomLifecycleMixin (room CRUD, timers)
│   │   ├── _helpers.py            # HelpersMixin (shared utilities)
│   │   ├── hooks.py               # HookEngine (sync/async pipelines)
│   │   ├── event_router.py        # EventRouter (broadcast + delivery)
│   │   ├── inbound_router.py      # InboundRoomRouter (message routing)
│   │   ├── transcoder.py          # DefaultContentTranscoder
│   │   ├── circuit_breaker.py     # CircuitBreaker (fault isolation)
│   │   ├── rate_limiter.py        # TokenBucketRateLimiter
│   │   ├── retry.py               # retry_with_backoff()
│   │   └── locks.py               # RoomLockManager + InMemoryLockManager
│   ├── channels/                  # Channel implementations
│   │   ├── __init__.py            # Factory functions (SMSChannel, EmailChannel, RCSChannel, etc.)
│   │   ├── base.py                # Channel ABC
│   │   ├── transport.py           # TransportChannel (unified generic channel)
│   │   ├── websocket.py           # WebSocketChannel
│   │   ├── ai.py                  # AIChannel (intelligence layer)
│   │   └── voice.py               # VoiceChannel (real-time audio with STT/TTS)
│   ├── models/                    # Pydantic data models
│   │   ├── enums.py               # All StrEnum types (16 enums)
│   │   ├── event.py               # RoomEvent, EventContent (9 content types)
│   │   ├── room.py                # Room, RoomTimers
│   │   ├── channel.py             # ChannelBinding, ChannelCapabilities, ChannelOutput, RateLimit, RetryPolicy
│   │   ├── hook.py                # HookResult, InjectedEvent
│   │   ├── identity.py            # Identity, IdentityResult, IdentityHookResult
│   │   ├── participant.py         # Participant
│   │   ├── delivery.py            # InboundMessage, InboundResult, ProviderResult, DeliveryResult, DeliveryStatus
│   │   ├── context.py             # RoomContext
│   │   ├── task.py                # Task, Observation
│   │   └── framework_event.py     # FrameworkEvent
│   ├── providers/                 # External service providers
│   │   ├── ai/                    # AIProvider ABC + MockAIProvider
│   │   ├── anthropic/             # AnthropicAIProvider + AnthropicConfig
│   │   ├── openai/                # OpenAIAIProvider + OpenAIConfig
│   │   ├── gemini/                # GeminiAIProvider + GeminiConfig
│   │   ├── pydantic_ai/           # Pydantic AI integration support
│   │   ├── sms/                   # SMSProvider ABC + MockSMSProvider + utilities
│   │   ├── voicemeup/             # VoiceMeUpSMSProvider + VoiceMeUpConfig + MMS aggregation
│   │   ├── twilio/                # TwilioSMSProvider + TwilioRCSProvider + configs
│   │   ├── telnyx/                # TelnyxSMSProvider + TelnyxRCSProvider + configs
│   │   ├── sinch/                 # SinchSMSProvider + SinchConfig
│   │   ├── rcs/                   # RCSProvider ABC + MockRCSProvider
│   │   ├── email/                 # EmailProvider ABC + MockEmailProvider
│   │   ├── elasticemail/          # ElasticEmailProvider + ElasticEmailConfig
│   │   ├── sendgrid/              # SendGridConfig (scaffolded)
│   │   ├── messenger/             # MessengerProvider ABC + FacebookMessengerProvider + MockMessengerProvider
│   │   ├── whatsapp/              # WhatsAppProvider ABC + MockWhatsAppProvider
│   │   └── http/                  # HTTPProvider ABC + WebhookHTTPProvider + MockHTTPProvider
│   ├── identity/                  # Identity resolution
│   │   ├── base.py                # IdentityResolver ABC
│   │   └── mock.py                # MockIdentityResolver
│   ├── realtime/                  # Ephemeral events
│   │   ├── base.py                # RealtimeBackend ABC
│   │   └── memory.py              # InMemoryRealtime
│   ├── voice/                     # Voice/audio support
│   │   ├── __init__.py            # Lazy loaders for optional providers
│   │   ├── base.py                # Shared types: VoiceSession, AudioChunk, callbacks
│   │   ├── events.py              # Voice event types (BargeIn, VAD, etc.)
│   │   ├── stt/                   # Speech-to-text providers
│   │   │   ├── base.py            # STTProvider ABC
│   │   │   ├── deepgram.py        # DeepgramSTTProvider + DeepgramConfig
│   │   │   └── mock.py            # MockSTTProvider
│   │   ├── tts/                   # Text-to-speech providers
│   │   │   ├── base.py            # TTSProvider ABC
│   │   │   ├── elevenlabs.py      # ElevenLabsTTSProvider + ElevenLabsConfig
│   │   │   └── mock.py            # MockTTSProvider
│   │   ├── backends/              # Voice transport backends
│   │   │   ├── base.py            # VoiceBackend ABC
│   │   │   ├── fastrtc.py         # FastRTCVoiceBackend + mount_fastrtc_voice
│   │   │   └── mock.py            # MockVoiceBackend + MockVoiceCall
│   │   └── realtime/              # Realtime voice (speech-to-speech)
│   │       ├── base.py            # RealtimeSession, RealtimeSessionState
│   │       ├── provider.py        # RealtimeVoiceProvider ABC
│   │       ├── transport.py       # RealtimeAudioTransport ABC
│   │       ├── ws_transport.py    # WebSocketRealtimeTransport
│   │       ├── fastrtc_transport.py # FastRTCRealtimeTransport + mount_fastrtc_realtime
│   │       ├── events.py          # Realtime voice events
│   │       └── mock.py            # MockRealtimeProvider + MockRealtimeTransport
│   └── store/                     # Persistence layer
│       ├── base.py                # ConversationStore ABC (30 abstract methods)
│       └── memory.py              # InMemoryStore implementation
├── tests/                         # Test suite
│   ├── conftest.py                # Shared fixtures
│   ├── test_channels/             # Channel-specific tests (7 files)
│   ├── test_providers/            # Provider-specific tests (15+ files)
│   ├── test_integration/          # End-to-end integration tests (8+ files)
│   └── test_*.py                  # Unit tests (~35 files)
├── site/                          # Built documentation output
├── pyproject.toml                 # Project configuration
├── Makefile                       # Development commands
├── mkdocs.yml                     # MkDocs configuration
├── uv.lock                        # Dependency lock file
├── .pre-commit-config.yaml        # Pre-commit hooks
├── AGENTS.md                      # AI coding assistant context
├── llms.txt                       # LLM context document
├── README.md                      # Project README
├── CONTRIBUTING.md                # Contribution guidelines
└── LICENSE                        # MIT license
```

---

## Data Models

All models inherit from `pydantic.BaseModel` with Pydantic v2 validation.

### Room

```python
class Room(BaseModel):
    id: str
    organization_id: str | None
    status: RoomStatus          # ACTIVE | PAUSED | CLOSED | ARCHIVED
    created_at: datetime
    updated_at: datetime
    closed_at: datetime | None
    timers: RoomTimers
    metadata: dict[str, Any]
    event_count: int
    latest_index: int
```

### RoomEvent

```python
class RoomEvent(BaseModel):
    id: str                     # UUID hex
    room_id: str
    type: EventType             # MESSAGE, SYSTEM, TYPING, etc.
    source: EventSource
    content: EventContent       # Discriminated union (see below)
    status: EventStatus         # PENDING | DELIVERED | READ | FAILED | BLOCKED
    blocked_by: str | None
    visibility: str             # "all" or channel-specific filter
    index: int
    chain_depth: int            # Reentry depth for loop prevention
    parent_event_id: str | None
    correlation_id: str | None
    idempotency_key: str | None
    created_at: datetime
    metadata: dict[str, Any]
    channel_data: ChannelData
    delivery_results: dict[str, Any]
```

### Content Types (Discriminated Union)

```python
EventContent = Annotated[
    TextContent          # Plain text with optional language
    | RichContent        # HTML/Markdown with buttons, cards, quick replies
    | MediaContent       # File attachment (URL, MIME type, caption)
    | LocationContent    # Lat/lng with label and address
    | AudioContent       # Audio file with optional transcript
    | VideoContent       # Video file with optional thumbnail
    | CompositeContent   # Multi-part combining multiple types (max depth 5)
    | SystemContent      # System messages with code and data
    | TemplateContent,   # Pre-approved templates (WhatsApp Business)
    Field(discriminator="type"),
]
```

### ChannelBinding

```python
class ChannelBinding(BaseModel):
    channel_id: str
    room_id: str
    channel_type: ChannelType
    category: ChannelCategory   # TRANSPORT | INTELLIGENCE
    direction: ChannelDirection  # INBOUND | OUTBOUND | BIDIRECTIONAL
    access: Access              # READ_WRITE | READ_ONLY | WRITE_ONLY | NONE
    muted: bool
    visibility: str
    participant_id: str | None
    last_read_index: int | None
    attached_at: datetime
    capabilities: ChannelCapabilities
    rate_limit: RateLimit | None
    retry_policy: RetryPolicy | None
    metadata: dict[str, Any]
```

### ChannelCapabilities

```python
class ChannelCapabilities(BaseModel):
    media_types: list[ChannelMediaType]
    max_length: int | None
    supports_threading: bool
    supports_reactions: bool
    supports_read_receipts: bool
    supports_typing: bool
    supports_templates: bool
    supports_rich_text: bool
    supports_buttons: bool
    max_buttons: int | None
    supports_cards: bool
    supports_quick_replies: bool
    supports_media: bool
    supported_media_types: list[str]
    max_media_size_bytes: int | None
    supports_audio: bool
    supports_video: bool
    delivery_mode: DeliveryMode  # BROADCAST | DIRECT | ROUND_ROBIN
```

### DeliveryStatus

```python
class DeliveryStatus(BaseModel):
    """Status update for an outbound message from a provider webhook."""
    provider: str               # e.g., "telnyx", "twilio"
    message_id: str             # Provider's unique message identifier
    status: str                 # e.g., "sent", "delivered", "failed"
    recipient: str              # Phone number/address sent to
    sender: str                 # Phone number/address sent from
    error_code: str | None      # Provider-specific error code
    error_message: str | None   # Human-readable error message
    timestamp: str | None       # When the status was reported
    raw: dict[str, Any]         # Original webhook payload
```

### AI Models

```python
class AITool(BaseModel):
    """Tool definition for function calling."""
    name: str
    description: str
    parameters: dict[str, Any]

class AIToolCall(BaseModel):
    """A tool call from the AI response."""
    id: str
    name: str
    arguments: dict[str, Any]

class AIContext(BaseModel):
    """Context passed to AI provider for generation."""
    messages: list[AIMessage]
    system_prompt: str | None
    temperature: float
    max_tokens: int
    tools: list[AITool]                 # Function calling definitions
    room: RoomContext | None
    target_capabilities: ChannelCapabilities | None
    target_media_types: list[ChannelMediaType]

class AIResponse(BaseModel):
    """Response from an AI provider."""
    content: str
    finish_reason: str | None
    usage: dict[str, int]
    tool_calls: list[AIToolCall]        # Function calls to execute
    tasks: list[Task]
    observations: list[Observation]
```

### Voice Models

```python
class VoiceSessionState(StrEnum):
    CONNECTING, ACTIVE, PAUSED, ENDED

class VoiceCapability(Flag):
    NONE = 0
    INTERRUPTION = auto()      # Can cancel audio playback
    PARTIAL_STT = auto()       # Provides partial transcription
    VAD_SILENCE = auto()       # Emits silence detection events
    VAD_AUDIO_LEVEL = auto()   # Emits audio level events
    BARGE_IN = auto()          # Detects user interrupts TTS

@dataclass
class VoiceSession:
    id: str
    room_id: str
    participant_id: str
    channel_id: str
    state: VoiceSessionState
    created_at: datetime
    metadata: dict[str, Any]       # Contains input/output sample rates

@dataclass
class AudioChunk:
    data: bytes
    sample_rate: int = 16000
    channels: int = 1
    format: str = "pcm_s16le"
    timestamp_ms: int | None = None
    is_final: bool = False

@dataclass(frozen=True)
class BargeInEvent:
    """User started speaking while TTS was playing."""
    session: VoiceSession
    interrupted_text: str
    audio_position_ms: int
    timestamp: datetime

@dataclass(frozen=True)
class TTSCancelledEvent:
    session: VoiceSession
    reason: Literal["barge_in", "explicit", "disconnect", "error"]
    text: str
    audio_position_ms: int
    timestamp: datetime
```

### Enums

| Enum | Values |
|---|---|
| `ChannelType` | `sms`, `mms`, `rcs`, `email`, `whatsapp`, `websocket`, `ai`, `voice`, `push`, `messenger`, `webhook`, `system` |
| `ChannelCategory` | `transport`, `intelligence` |
| `ChannelDirection` | `inbound`, `outbound`, `bidirectional` |
| `ChannelMediaType` | `text`, `rich`, `media`, `audio`, `video`, `location`, `template` |
| `EventType` | `message`, `system`, `typing`, `read_receipt`, `delivery_receipt`, `presence`, `reaction`, `edit`, `delete`, `participant_joined`, `participant_left`, `participant_identified`, `channel_attached`, `channel_detached`, `channel_muted`, `channel_unmuted`, `channel_updated`, `task_created`, `observation` |
| `EventStatus` | `pending`, `delivered`, `read`, `failed`, `blocked` |
| `RoomStatus` | `active`, `paused`, `closed`, `archived` |
| `Access` | `read_write`, `read_only`, `write_only`, `none` |
| `IdentificationStatus` | `identified`, `pending`, `ambiguous`, `unknown`, `challenge_sent`, `rejected` |
| `ParticipantRole` | `owner`, `agent`, `member`, `observer`, `bot` |
| `ParticipantStatus` | `active`, `inactive`, `left`, `banned` |
| `TaskStatus` | `pending`, `in_progress`, `completed`, `failed`, `cancelled` |
| `DeliveryMode` | `broadcast`, `direct`, `round_robin` |
| `HookTrigger` | `before_broadcast`, `after_broadcast`, `on_channel_attached`, `on_channel_detached`, `on_channel_muted`, `on_channel_unmuted`, `on_room_created`, `on_room_paused`, `on_room_closed`, `on_identity_ambiguous`, `on_identity_unknown`, `on_participant_identified`, `on_task_created`, `on_error`, `on_delivery_status`, `on_speech_start`, `on_speech_end`, `on_transcription`, `before_tts`, `after_tts`, `on_barge_in`, `on_tts_cancelled`, `on_partial_transcription`, `on_vad_silence`, `on_vad_audio_level` |
| `HookExecution` | `sync`, `async` |

---

## API Design

### Channel ABC

Every channel implements this interface:

```python
class Channel(ABC):
    channel_type: ChannelType
    category: ChannelCategory
    direction: ChannelDirection

    @abstractmethod
    async def handle_inbound(self, message: InboundMessage, context: RoomContext) -> RoomEvent:
        """Convert an inbound message into a RoomEvent."""

    @abstractmethod
    async def deliver(self, event: RoomEvent, binding: ChannelBinding, context: RoomContext) -> ChannelOutput:
        """Push an event to the external system."""

    async def on_event(self, event: RoomEvent, binding: ChannelBinding, context: RoomContext) -> ChannelOutput:
        """React to an event (default: no-op for transport channels)."""

    def capabilities(self) -> ChannelCapabilities:
        """Declare supported media types and features."""

    async def close(self) -> None:
        """Close the channel and its provider."""
```

### Voice ABCs

```python
class VoiceBackend(ABC):
    """Audio transport: WebSocket/WebRTC connections, VAD, session management."""

    @property
    @abstractmethod
    def name(self) -> str: ...

    @property
    def capabilities(self) -> VoiceCapability: ...

    @abstractmethod
    async def connect(self, room_id, participant_id, channel_id, *, metadata=None) -> VoiceSession: ...

    @abstractmethod
    async def disconnect(self, session: VoiceSession) -> None: ...

    @abstractmethod
    def on_speech_start(self, callback: SpeechStartCallback) -> None: ...

    @abstractmethod
    def on_speech_end(self, callback: SpeechEndCallback) -> None: ...

    @abstractmethod
    async def send_audio(self, session, audio: bytes | AsyncIterator[AudioChunk]) -> None: ...

    async def send_transcription(self, session, text, role="user") -> None: ...
    async def cancel_audio(self, session) -> bool: ...
    def is_playing(self, session) -> bool: ...

    # Enhanced callbacks (opt-in via capabilities)
    def on_partial_transcription(self, callback) -> None: ...
    def on_vad_silence(self, callback) -> None: ...
    def on_vad_audio_level(self, callback) -> None: ...
    def on_barge_in(self, callback) -> None: ...


class STTProvider(ABC):
    """Speech-to-text: transcribe audio to text."""

    @abstractmethod
    async def transcribe(self, audio: AudioContent | AudioChunk) -> str: ...

    async def transcribe_stream(self, audio_stream) -> AsyncIterator[TranscriptionResult]: ...
    async def close(self) -> None: ...


class TTSProvider(ABC):
    """Text-to-speech: synthesize text to audio."""

    @abstractmethod
    async def synthesize(self, text, *, voice=None) -> AudioContent: ...

    async def synthesize_stream(self, text, *, voice=None) -> AsyncIterator[AudioChunk]: ...
    async def close(self) -> None: ...
```

### RoomKit Public API

The `RoomKit` orchestrator exposes these public methods, organized by mixin:

```python
class RoomKit(InboundMixin, ChannelOpsMixin, RoomLifecycleMixin, HelpersMixin):
    def __init__(
        self,
        store=None,                    # ConversationStore (default: InMemoryStore)
        identity_resolver=None,        # IdentityResolver
        identity_channel_types=None,   # Restrict identity resolution to channel types
        inbound_router=None,           # InboundRoomRouter
        lock_manager=None,             # RoomLockManager (default: InMemoryLockManager)
        realtime=None,                 # RealtimeBackend (default: InMemoryRealtime)
        max_chain_depth=5,             # Max AI reentry depth
        identity_timeout=10.0,         # Timeout for identity resolution (seconds)
        process_timeout=30.0,          # Timeout for locked processing (seconds)
    )

    # --- Properties ---
    @property
    def store(self) -> ConversationStore
    @property
    def hook_engine(self) -> HookEngine
    @property
    def realtime(self) -> RealtimeBackend

    # --- Inbound processing (InboundMixin) ---
    async def process_inbound(self, message: InboundMessage) -> InboundResult

    # --- Webhook handling ---
    async def process_webhook(self, meta: WebhookMeta, channel_id: str) -> None
    async def process_delivery_status(self, status: DeliveryStatus) -> None

    # --- Channel management (ChannelOpsMixin) ---
    def register_channel(self, channel: Channel) -> None
    async def attach_channel(self, room_id, channel_id, channel_type=None,
                             category=TRANSPORT, access=READ_WRITE,
                             visibility="all", **kwargs) -> ChannelBinding
    async def detach_channel(self, room_id, channel_id) -> bool
    async def mute(self, room_id, channel_id) -> ChannelBinding
    async def unmute(self, room_id, channel_id) -> ChannelBinding
    async def set_visibility(self, room_id, channel_id, visibility) -> ChannelBinding
    async def set_access(self, room_id, channel_id, access) -> ChannelBinding
    async def update_binding_metadata(self, room_id, channel_id, metadata) -> ChannelBinding
    def get_channel(self, channel_id) -> Channel | None
    def list_channels(self) -> list[Channel]
    async def get_binding(self, room_id, channel_id) -> ChannelBinding
    async def list_bindings(self, room_id) -> list[ChannelBinding]

    # --- Room lifecycle (RoomLifecycleMixin) ---
    async def create_room(self, room_id=None, metadata=None) -> Room
    async def get_room(self, room_id) -> Room
    async def close_room(self, room_id) -> Room
    async def check_room_timers(self, room_id) -> Room
    async def check_all_timers(self) -> list[Room]
    async def update_room_metadata(self, room_id, metadata) -> Room
    async def ensure_participant(self, room_id, channel_id, participant_id,
                                 display_name=None) -> Participant
    async def resolve_participant(self, room_id, participant_id, identity_id,
                                  resolved_by="manual") -> Participant

    # --- Direct send ---
    async def send_event(self, room_id, channel_id, content,
                         event_type=MESSAGE, chain_depth=0) -> RoomEvent

    # --- Queries ---
    async def get_timeline(self, room_id, offset=0, limit=50,
                           visibility_filter=None) -> list[RoomEvent]
    async def list_tasks(self, room_id, status=None) -> list[Task]
    async def list_observations(self, room_id) -> list[Observation]

    # --- WebSocket lifecycle ---
    async def connect_websocket(self, channel_id, connection_id, send_fn) -> None
    async def disconnect_websocket(self, channel_id, connection_id) -> None

    # --- Read tracking ---
    async def mark_read(self, room_id, channel_id, event_id) -> None
    async def mark_all_read(self, room_id, channel_id) -> None

    # --- Realtime (ephemeral events) ---
    async def publish_typing(self, room_id, user_id, is_typing=True) -> None
    async def publish_presence(self, room_id, user_id, status) -> None
    async def publish_read_receipt(self, room_id, user_id, event_id) -> None
    async def subscribe_room(self, room_id, callback) -> str
    async def unsubscribe_room(self, subscription_id) -> bool

    # --- Hook registration ---
    def hook(self, trigger, execution=SYNC, priority=0, name="", timeout=30.0,
             channel_types=None, channel_ids=None, directions=None) -> decorator
    def on(self, event_type: str) -> decorator      # Framework event handler
    def identity_hook(self, trigger, channel_types=None, channel_ids=None,
                       directions=None) -> decorator    # Identity resolution hook
    def on_delivery_status(self, fn) -> decorator   # Delivery status handler
    def add_room_hook(self, room_id, trigger, execution, fn, priority=0, name="") -> None
    def remove_room_hook(self, room_id, name) -> bool

    # --- Voice ---
    async def connect_voice(self, room_id, participant_id, channel_id, metadata=None) -> VoiceSession
    async def disconnect_voice(self, session: VoiceSession) -> None
    async def transcribe(self, audio: AudioContent) -> str
    async def synthesize(self, text: str, voice=None) -> AudioContent

    # --- Context manager ---
    async def close(self) -> None
    async def __aenter__(self) -> RoomKit
    async def __aexit__(self, ...) -> None
```

### ConversationStore ABC

The storage interface defines 30 abstract methods across 8 categories:

| Category | Methods |
|---|---|
| **Rooms** | `create_room`, `get_room`, `update_room`, `delete_room`, `list_rooms`, `find_rooms`, `find_latest_room`, `find_room_id_by_channel` |
| **Events** | `add_event`, `get_event`, `list_events`, `check_idempotency`, `get_event_count` |
| **Bindings** | `add_binding`, `get_binding`, `update_binding`, `remove_binding`, `list_bindings` |
| **Participants** | `add_participant`, `get_participant`, `update_participant`, `list_participants` |
| **Identities** | `create_identity`, `get_identity`, `resolve_identity`, `link_address` |
| **Tasks** | `add_task`, `get_task`, `list_tasks`, `update_task` |
| **Observations** | `add_observation`, `list_observations` |
| **Read Tracking** | `mark_read`, `mark_all_read`, `get_unread_count` |

---

## Configuration Management

### Provider Configuration

All provider configs use Pydantic models with `SecretStr` for sensitive values:

```python
class AnthropicConfig(BaseModel):
    api_key: SecretStr
    model: str = "claude-sonnet-4-20250514"
    max_tokens: int = 1024

class OpenAIConfig(BaseModel):
    api_key: SecretStr
    model: str = "gpt-4o"
    max_tokens: int = 1024

class GeminiConfig(BaseModel):
    api_key: SecretStr
    model: str = "gemini-2.0-flash"
    max_tokens: int = 1024
    temperature: float = 1.0

class TelnyxConfig(BaseModel):
    api_key: SecretStr
    from_number: str
    messaging_profile_id: str | None = None
    timeout: float = 10.0

class TelnyxRCSConfig(BaseModel):
    api_key: SecretStr
    agent_id: str
    messaging_profile_id: str | None = None
    timeout: float = 10.0

class TwilioConfig(BaseModel):
    account_sid: str
    auth_token: SecretStr
    from_number: str

class VoiceMeUpConfig(BaseModel):
    username: str
    auth_token: SecretStr
    from_number: str
    base_url: str = "https://api.voicemeup.com/"
    timeout: float = 10.0

class ElasticEmailConfig(BaseModel):
    api_key: SecretStr
    from_email: str
    from_name: str | None = None
```

### RoomKit Initialization

The orchestrator accepts pluggable components via constructor arguments:

```python
kit = RoomKit(
    store=InMemoryStore(),              # Default
    identity_resolver=MyResolver(),     # Optional
    identity_channel_types={ChannelType.SMS},  # Restrict identity to SMS only
    inbound_router=MyRouter(),          # Default: DefaultInboundRoomRouter
    lock_manager=InMemoryLockManager(), # Default
    realtime=InMemoryRealtime(),        # Default
    max_chain_depth=5,                  # Default
    identity_timeout=10.0,              # Default (seconds)
    process_timeout=30.0,               # Default (seconds)
)
```

### Optional Dependencies

Dependencies are lazily loaded. Each provider group has its own optional extra:

```toml
[project.optional-dependencies]
httpx = ["httpx>=0.27"]
anthropic = ["anthropic>=0.30"]
openai = ["openai>=1.30"]
gemini = ["google-genai>=1.0.0"]
twilio = ["twilio>=9.0"]
phonenumbers = ["phonenumbers>=8.13"]
pynacl = ["pynacl>=1.5"]
pydantic-ai = ["pydantic-ai>=0.1"]
fastrtc = ["fastrtc", "numpy"]
websocket = ["websockets>=13.0"]
sse = ["httpx>=0.27", "httpx-sse>=0.4"]
providers = ["roomkit[httpx,anthropic,openai,gemini,twilio]"]
all = ["roomkit[providers,pydantic-ai,phonenumbers,pynacl]"]
```

---

## Build and Deployment

### Build System

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.version]
path = "src/roomkit/_version.py"

[tool.hatch.build.targets.wheel]
packages = ["src/roomkit"]
artifacts = ["*.md", "*.txt"]

[tool.hatch.build.targets.wheel.force-include]
"AGENTS.md" = "roomkit/AGENTS.md"
"llms.txt" = "roomkit/llms.txt"
```

### Development Commands (Makefile)

| Command | Action |
|---|---|
| `make install` | `uv sync --extra dev` |
| `make lint` | `uv run ruff check src/ tests/` |
| `make format` | `uv run ruff format src/ tests/` |
| `make typecheck` | `uv run mypy src/roomkit/` |
| `make test` | `uv run pytest` |
| `make coverage` | `uv run pytest --cov=roomkit --cov-report=term-missing` |
| `make all` | `lint + typecheck + test` |
| `make clean` | Remove build artifacts, caches |

### CI/CD Pipeline (GitHub Actions)

The CI pipeline runs on push to `main` and on pull requests:

```yaml
jobs:
  lint:        # Ruff check + format check (Python 3.12)
  typecheck:   # mypy strict (Python 3.12)
  test:        # pytest with coverage (Python 3.12 + 3.13 matrix)
```

Coverage reports are uploaded as artifacts for the Python 3.12 run.

---

## Testing Strategy

### Test Organization

| Category | Path | Scope |
|---|---|---|
| **Unit** | `tests/test_*.py` | Individual components (models, enums, hooks, store, locks, circuit breakers, retry, rate limiting, public API) |
| **Channel** | `tests/test_channels/` | Channel implementations (websocket, sms, email, messenger, whatsapp, ai, http) |
| **Provider** | `tests/test_providers/` | Provider integrations (anthropic, openai, gemini, voicemeup, twilio, telnyx, sinch, rcs, elasticemail, http, messenger) |
| **Voice** | `tests/test_voice*.py`, `tests/test_fastrtc_realtime_transport.py` | Voice subsystem: STT/TTS mocks, VoiceBackend, VoiceChannel pipeline, FastRTCVoiceBackend, mu-law encoding, FastRTCRealtimeTransport |
| **Integration** | `tests/test_integration/` | Cross-component workflows (quickstart, AI assistant, human-AI, cross-channel, dynamic channels, observer, identity resolution, AI chain depth) |

### Test Configuration

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"           # All async tests run automatically
filterwarnings = ["error"]      # All warnings treated as errors

[tool.coverage.report]
show_missing = true
fail_under = 90                 # 90% minimum coverage required
```

### Testing Patterns

- **Mock providers** ship with the library (`MockAIProvider`, `MockSMSProvider`, `MockEmailProvider`, `MockHTTPProvider`, `MockMessengerProvider`, `MockWhatsAppProvider`, `MockRCSProvider`, `MockIdentityResolver`, `MockSTTProvider`, `MockTTSProvider`, `MockVoiceBackend`)
- **Shared fixtures** in `conftest.py` provide `InMemoryStore`, pre-built rooms, and event/binding factory helpers (`make_event()`, `make_binding()`)
- **All tests are async** using `pytest-asyncio` with `asyncio_mode="auto"`
- **Integration tests** cover multi-channel flows, AI chain depth limiting, identity resolution, cross-channel delivery, and dynamic channel attachment

### Key Test Scenarios

- Room lifecycle (create, pause, close with timers)
- Hook pipelines (sync block/allow/modify, async side effects, priority ordering, timeout handling, filtering)
- Cross-channel message routing and content transcoding
- AI channel reentry with chain depth limits
- AI per-room configuration and function calling
- Identity resolution with ambiguity handling and challenge flows
- Concurrent inbound processing with per-room locks
- Circuit breaker state transitions (closed/open/half-open)
- Rate limiter token refill behavior and wait semantics
- Idempotency key deduplication (checked inside room lock)
- Muted channel behavior (events received, responses suppressed, tasks/observations preserved)
- Visibility filtering (all, none, transport, intelligence, specific channel IDs)
- Framework event emission for observability
- RCS provider functionality and SMS fallback
- MMS aggregation (VoiceMeUp split webhooks)
- Delivery status webhook handling
- Voice pipeline: speech end → STT → hooks → inbound → AI → TTS → audio
- VoiceBackend session lifecycle and callback registration
- FastRTCVoiceBackend: session management, mu-law encoding, WebSocket audio send
- FastRTCRealtimeTransport: session/handler mapping, audio queueing, DataChannel messaging, disconnect cleanup
- Barge-in detection and TTS cancellation
- Voice capability flags and enhanced callbacks

---

## Code Quality Tools

### Ruff (Linter + Formatter)

```toml
[tool.ruff]
target-version = "py312"
line-length = 99
src = ["src", "tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM"]
```

Rules enabled:
- **E** -- pycodestyle errors
- **F** -- pyflakes
- **I** -- isort (import sorting)
- **N** -- pep8-naming
- **UP** -- pyupgrade
- **B** -- flake8-bugbear
- **SIM** -- flake8-simplify

### mypy (Type Checker)

```toml
[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true

[[tool.mypy.overrides]]
module = ["google", "google.genai", "google.genai.*"]
ignore_missing_imports = true
```

The library is **PEP 561 compliant** (`py.typed` marker) and fully type-safe under strict mypy checking.

### Pre-commit Hooks

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    hooks:
      - id: ruff          # Lint check with --fix
      - id: ruff-format   # Format check
  - repo: https://github.com/pre-commit/mirrors-mypy
    hooks:
      - id: mypy          # Strict type check
```

---

## Development Environment Setup

```bash
# Clone the repository
git clone https://github.com/roomkit-live/roomkit.git
cd roomkit

# Install all dev dependencies (requires uv)
make install
# or: uv sync --extra dev

# Run the full quality gate
make all
# or individually:
make lint        # Check linting
make format      # Auto-format code
make typecheck   # Strict type checking
make test        # Run test suite
make coverage    # Coverage report (90% threshold)

# Run examples
uv run python examples/quickstart.py

# Build documentation
uv run mkdocs serve    # Local dev server
uv run mkdocs build    # Build to site/
```

---

## Key Architectural Patterns

### Mixin Composition

The `RoomKit` class uses multiple inheritance to separate concerns while maintaining a single public interface:

```python
class RoomKit(InboundMixin, ChannelOpsMixin, RoomLifecycleMixin, HelpersMixin):
    """Central orchestrator tying rooms, channels, hooks, and storage."""
```

Each mixin declares the shared state it requires via class-level type annotations (e.g., `_store: ConversationStore`), and the `RoomKit.__init__` initializes all shared state.

### Data-Driven Transport Channels

Instead of subclassing `Channel` for each transport type, a single `TransportChannel` class handles all transports via configuration:

```python
def SMSChannel(channel_id, *, provider=None, from_number=None):
    return TransportChannel(
        channel_id,
        ChannelType.SMS,
        provider=provider,
        capabilities=SMS_CAPABILITIES,
        recipient_key="phone_number",
        defaults={"from_": from_number},
    )

def RCSChannel(channel_id, *, provider=None):
    return TransportChannel(
        channel_id,
        ChannelType.RCS,
        provider=provider,
        capabilities=RCS_CAPABILITIES,
        recipient_key="phone_number",
    )
```

The `recipient_key` determines which binding metadata field contains the delivery address. `defaults` with `None` values are resolved from binding metadata at delivery time.

### Discriminated Union Content

Event content uses Pydantic's discriminator pattern for type-safe polymorphism without `isinstance` checks:

```python
EventContent = Annotated[
    TextContent | RichContent | MediaContent | ...,
    Field(discriminator="type"),
]
```

Each content type has a `type` literal field (e.g., `type: Literal["text"] = "text"`).

### Side Effects Model

Hooks and intelligence channels can produce three types of side effects:

- **InjectedEvent** -- Synthetic events delivered to specific channels (e.g., challenge messages)
- **Task** -- Work items with status tracking for human follow-up
- **Observation** -- Intelligence findings with category and confidence score

All side effects are collected during the pipeline and persisted after broadcast completes.

### Per-Room AI Configuration

AI channels support per-room configuration via binding metadata:

```python
await kit.attach_channel("room-1", "ai-bot",
    category=ChannelCategory.INTELLIGENCE,
    metadata={
        "system_prompt": "You are a legal assistant.",
        "temperature": 0.3,
        "max_tokens": 2048,
        "tools": [
            {
                "name": "search_cases",
                "description": "Search legal case database",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "query": {"type": "string"},
                        "jurisdiction": {"type": "string"},
                    },
                },
            },
        ],
    },
)
```

---

## Known Technical Debt and Recommendations

### Partially Implemented Providers

The following providers have implementations but may need enhancement:

- **WhatsApp** -- Mock provider only; no production WhatsApp Business API integration
- **SendGrid Email** -- `SendGridConfig` exists, provider not yet implemented

### Storage Layer

- **InMemoryStore only** -- No persistent storage backend ships with the library. Production deployments need a custom `ConversationStore` implementation (PostgreSQL, Redis, etc.).
- **No migration tooling** -- Schema changes to the store interface require manual migration handling.

### Missing Infrastructure Patterns

- **No distributed locking** -- `InMemoryLockManager` uses `asyncio.Lock`, suitable for single-process deployments only. Multi-process deployments need Redis-based or similar distributed locks.
- **No distributed realtime** -- `InMemoryRealtime` is single-process only. Multi-process deployments need Redis pub/sub or similar.
- **No event bus** -- Events are broadcast in-process. Cross-service event distribution would require an external message broker integration.
- **No metrics/tracing** -- The `FrameworkEvent` system provides observability hooks, but there is no built-in integration with OpenTelemetry or similar tracing frameworks.

### Placeholder Channel Types

- All source files stay under 500 lines, maintaining readability
- The `__init__.py` exports a large public surface (~150 symbols) -- users can also use sub-module imports (e.g., `from roomkit.channels import SMSChannel`) for more granular control
- The `ChannelType` enum includes `push` and `system` values that have no channel implementations yet (voice is now implemented via `VoiceChannel`)
