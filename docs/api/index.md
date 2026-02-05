# API Reference

Full auto-generated reference for the `roomkit` public API.

## Core

- **[RoomKit](roomkit.md)** — Central orchestrator for rooms, channels, hooks, and storage
- **[Hooks](hooks.md)** — Hook engine, registrations, and pipeline results
- **[Routing](routing.md)** — Inbound room routing
- **[Store](store.md)** — Conversation storage ABC and in-memory implementation
- **[Realtime](realtime.md)** — Ephemeral events (typing, presence, read receipts)
- **[Sources](sources.md)** — Event-driven message ingestion (WebSocket, NATS, etc.)

## Channels

- **[Channel ABC](channel.md)** — Base class for all channels
- **[Built-in Channels](channels.md)** — SMS, RCS, Email, AI, WebSocket, Voice, WhatsApp, Messenger, HTTP
- **[Voice Channel](providers-voice.md)** — VoiceBackend, STTProvider, TTSProvider, voice events

## Models

- **[Room & Timers](room.md)** — Room model and timer configuration
- **[Events & Content](events.md)** — RoomEvent, content types, EventSource
- **[Channel Models](channel-models.md)** — ChannelBinding, ChannelCapabilities, ChannelOutput, RateLimit, RetryPolicy
- **[Participants & Identity](identity.md)** — Participant, Identity, IdentityResult
- **[Delivery](delivery.md)** — InboundMessage, InboundResult, DeliveryResult, ProviderResult
- **[Hooks & Side Effects](hook-models.md)** — HookResult, InjectedEvent, Task, Observation
- **[Enums](enums.md)** — All enumerations

## Providers

- **[AI](providers-ai.md)** — AIProvider, AIContext, AIResponse
- **[SMS](providers-sms.md)** — SMSProvider, Sinch, Telnyx, Twilio, VoiceMeUp
- **[RCS](providers-rcs.md)** — RCSProvider, Telnyx RCS, Twilio RCS
- **[Email](providers-email.md)** — EmailProvider, ElasticEmail
- **[WhatsApp](providers-whatsapp.md)** — WhatsAppProvider
- **[Messenger](providers-messenger.md)** — MessengerProvider, Facebook
- **[HTTP](providers-http.md)** — HTTPProvider, Webhook
