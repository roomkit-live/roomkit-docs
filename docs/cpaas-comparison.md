# CPaaS Conversation API Comparison & RFC Gap Analysis

**Date:** 2026-01-27
**Purpose:** Compare major CPaaS conversation APIs to validate and strengthen our framework RFC.

---

## 1. Platforms Studied

| Platform | API Name | Architecture Style |
|---|---|---|
| **Twilio** | Conversations API | Conversation-centric, multi-party, SDK + webhook |
| **Sinch** | Conversation API | App → Contact → Conversation, unified send, webhook-only |
| **Vonage** | Conversation API + Messages API | Event-sourced, multi-modal (voice+video+text) |
| **MessageBird (Bird)** | Conversations API | Contact-centric, simple unified API |
| **Infobip** | Messages API + Conversations API | Enterprise contact center with CDP |

---

## 2. Concept Mapping

### How each platform maps to our RFC concepts

| Our RFC Concept | Twilio | Sinch | Vonage | MessageBird | Infobip |
|---|---|---|---|---|---|
| **Room** | Conversation | Conversation | Conversation | Conversation | Conversation |
| **Channel** | Participant (carries binding) | Channel identity on Contact | Member + Leg | Channel (resource) | Channel field on message |
| **ChannelBinding** | Participant with MessagingBinding | channel_identities on Contact | Member (INVITED→JOINED→LEFT) | Implicit per conversation | Implicit |
| **Permission (access/mute/visibility)** | None (all participants equal) | None | None | None | Agent roles only |
| **Provider** | Twilio IS the provider | Sinch IS the provider | Vonage IS the provider | Bird IS the provider | Infobip IS the provider |
| **Hook (sync)** | Pre-event webhook (can block/modify) | None | None | None | None |
| **Hook (async)** | Post-event webhook | Webhook triggers | Webhook/Event | Webhook | Webhook |
| **EventContent** | Body + Media + ContentSid | Message types (text/media/card/carousel/...) | Event body | Content object (type + payload) | Content body |
| **EventSource** | Source field (SDK/API/SMS/WHATSAPP) | channel_properties + raw data | Event metadata | Channel-specific fields | Channel + raw payload |
| **Channel metadata** | Attributes (32KB JSON on everything) | metadata + metadata_json at 3 levels | properties + custom_data | Limited customDetails | People custom attributes + callbackData |
| **Identity** | User resource (chat only), no cross-channel merge | Contact with channel_identities (cross-channel) | User resource | Contact with identifiers (cross-channel) | People CDP (full) |
| **Side effects (tasks/observations)** | None | None | Custom events (custom:*) | None | Internal notes |
| **AI Channel** | None | None | None | None | None |
| **MCP Server** | None | None | None | None | None |

### Key insight: No CPaaS has our permission system, AI-as-channel, or two output paths. These are our differentiators.

---

## 3. Detailed Feature Comparison

### 3.1 Conversation Lifecycle

| Feature | Twilio | Sinch | Vonage | MessageBird | Infobip | **Our RFC** |
|---|---|---|---|---|---|---|
| States | active → inactive → closed | Active only (stop endpoint) | No formal states | active / archived | OPEN → WAITING → SOLVED → CLOSED | active, paused, closed, archived |
| Auto-transitions (timers) | **Yes** (TimersInactive, TimersClosed) | No | No | No | SLA timers | **No** |
| Reopen closed | Yes | No | N/A | Yes | Yes | Not specified |
| Auto-create on inbound | Yes | Yes | Yes | Yes | Yes | Hook-based |

**Gap identified: We should add timer-based auto-transitions.** Twilio's pattern is elegant:
- `timers.inactive`: auto-transition from active → paused after N inactivity
- `timers.closed`: auto-transition from paused → closed after N more inactivity

### 3.2 Channel Management

| Feature | Twilio | Sinch | Vonage | MessageBird | Infobip | **Our RFC** |
|---|---|---|---|---|---|---|
| Add channel at runtime | Add Participant | Implicit on send | Add Member | Implicit | Implicit | **attach_channel()** |
| Remove channel at runtime | Delete Participant | N/A | Member LEFT | N/A | N/A | **detach_channel()** |
| Channel permissions | All equal | All equal | All equal | All equal | Agent roles | **access + mute + visibility** |
| Dynamic permission changes | No | No | No | No | No | **Yes (set_access, mute, set_visibility)** |
| Channel fallback | Via Messaging Service | **channel_priority_order** | No | **Automatic with timers** | Via automation | Not yet |
| Channel as first-class resource | No (implicit in Participant) | No (implicit in Contact) | No (implicit in Member) | **Yes (Channel resource with ID)** | No | **Yes (Channel registered with ID)** |

**Gap identified: Channel fallback.** Sinch and MessageBird both have automatic fallback. If delivery via WhatsApp fails, try SMS. We should add this:
- `ChannelBinding.fallback_channel_id: str | None`
- `ChannelBinding.fallback_after: timedelta | None`
- Or: a `FallbackPolicy` on the Room or as a hook

### 3.3 Message / Content Model

| Content Type | Twilio | Sinch | Vonage | MessageBird | Infobip | **Our RFC** |
|---|---|---|---|---|---|---|
| Text | Yes | Yes | Yes | Yes | Yes | TextContent |
| Image/Media | Yes (Media SID) | media_message | Yes | image/video/audio/file | Yes | MediaContent |
| Rich cards | Via Content Templates | card_message | No | interactive | Yes | RichContent |
| Carousel | Via Content Templates | carousel_message | No | interactive | carousel | RichContent |
| Buttons/Quick replies | Via Content Templates | choice_message | No | interactive | interactive | RichContent |
| Location | No | location_message | No | location | location | LocationContent |
| Contact/vCard | No | contact_info_message | No | No | contact | Not yet |
| List picker | No | list_message | No | No | Yes | Not yet |
| Templates (WhatsApp HSM) | Content Templates (ContentSid) | template_message | template | hsm | template | Not yet |
| Audio (voice note) | Media attachment | media_message | audio event | audio | audio | AudioContent (Phase 4+) |
| Video | Media attachment | media_message | video event | video | video | VideoContent (Phase 4+) |
| Custom/extensible | Attributes on message | No | **custom:* events** | No | No | EventType.custom |
| Auto-transcoding | No (Content Templates handle it) | **Yes (automatic per channel)** | No | No | No | Capability-aware AI generation |

**Gap identified: Templates.** WhatsApp requires pre-approved templates for outbound (outside 24h window). This is a hard requirement for WhatsApp support. We need a `TemplateContent` type.

**Gap identified: Auto-transcoding for non-AI paths.** When a human on WhatsApp sends a rich card to an SMS user, what happens? Sinch auto-transcodes. Our RFC handles this for AI (capability-aware generation) but not for human-to-human cross-channel.

### 3.4 Identity & Cross-Channel Resolution

| Feature | Twilio | Sinch | Vonage | MessageBird | Infobip | **Our RFC** |
|---|---|---|---|---|---|---|
| Identity concept | User (chat only) | Contact + channel_identities | User | Contact + identifiers | People (CDP) | Identity |
| Cross-channel linking | No (manual) | **Yes (Contact has multiple channels)** | Manual | **Yes (Contact identifiers)** | **Yes (People module)** | IdentityResolver ABC |
| Contact merge | No | **Yes (merge API)** | No | Manual | Yes | Not yet |
| Auto-creation | No | Yes (on inbound) | No | Yes (on inbound) | Yes | Hook-based |
| External ID linking | Via Attributes | external_id field | Via custom_data | Via customDetails | Via custom attributes | external_id on Identity |
| Contact profiles per channel | No | channel profiles endpoint | No | No | Per-channel info | channel_addresses on Identity |

**Gap identified: Contact merge API.** When you discover two identities are the same person, you need an explicit merge. Sinch has `contacts:merge`. We should add `fw.merge_identities(source_id, destination_id)`.

### 3.5 Webhooks / Events / Real-time

| Feature | Twilio | Sinch | Vonage | MessageBird | Infobip | **Our RFC** |
|---|---|---|---|---|---|---|
| Pre-event hooks (can block) | **Yes (PreWebhookUrl)** | No | No | No | No | **Yes (sync hooks)** |
| Post-event hooks | **Yes (PostWebhookUrl)** | **Yes (webhook triggers)** | **Yes** | **Yes** | **Yes** | **Yes (async hooks)** |
| Event granularity | 12+ event types | 17+ trigger types | Rich (custom events) | Basic (created/updated) | Moderate | 16+ event types |
| WebSocket / real-time | **Yes (SDK)** | No | **Yes (Client SDK)** | No | Partial (live chat widget) | **Yes (WebSocketChannel)** |
| Typing indicators | **Yes (built-in)** | **Yes (composing events)** | **Yes (typing on/off)** | No | Agent typing only | **Yes (EventType.typing)** |
| Read receipts | **Yes (per-participant index)** | **Yes (via delivery status)** | Yes | Yes (status=read) | Yes (SEEN status) | **Yes (EventType.read_receipt)** |
| Delivery status tracking | Yes (per participant) | Yes (webhook) | Yes | Yes (message status) | Yes (very granular) | Yes (EventType.delivery_status) |
| Sequential indexing | **Yes (Message.Index)** | No (timestamp-based) | **Yes (Event.id sequential)** | No | No | **Yes (RoomEvent.index)** |
| Read horizon tracking | **Yes (lastReadMessageIndex per participant)** | No | No | No | No | **Yes (ChannelBinding.last_read_index)** |

**Resolved (v7+): Sequential message indexing.** Adopted from Twilio's `Message.Index`. Every `RoomEvent` has a monotonically increasing `index` (0, 1, 2...) per room. Enables pagination, gap detection, and read tracking.

**Resolved (v7+): Read horizon per participant.** Adopted from Twilio's `lastReadMessageIndex`. Tracked via `ChannelBinding.last_read_index`. Enables "unread count" UIs.

### 3.6 Metadata

| Feature | Twilio | Sinch | Vonage | MessageBird | Infobip | **Our RFC** |
|---|---|---|---|---|---|---|
| Conversation metadata | **Attributes (32KB JSON)** | metadata + metadata_json | properties + custom_data | Limited | Tags, topic, notes | **metadata: dict** |
| Participant/Channel metadata | **Attributes (32KB JSON)** | N/A | Member metadata | Contact customDetails | People custom attrs | **ChannelBinding.metadata** |
| Message metadata | **Attributes (32KB JSON)** | message_metadata (string) | Event body (any) | Limited | callbackData | **RoomEvent.metadata** |
| Correlation ID | No (use Attributes) | **correlation_id** | No | No | **callbackData** | **Yes (RoomEvent.correlation_id)** |
| Mutable | Yes (all levels) | Yes (all levels) | Yes | Yes | Yes | **Yes (all levels)** |

**Resolved (v7+): Correlation ID.** Adopted from Sinch and Infobip. `RoomEvent.correlation_id: str | None` links messages to external systems (e.g., integrator's ticket ID). Echoed in delivery reports.

### 3.7 Advanced Features

| Feature | Twilio | Sinch | Vonage | MessageBird | Infobip | **Our RFC** |
|---|---|---|---|---|---|---|
| Multi-party conversations | **Yes (N participants)** | No (1:1 only) | **Yes (N members)** | No (1:1) | Agent-customer | **Yes (N channels)** |
| AI as participant | No | No | No | No | No | **Yes (AIChannel)** |
| Permission control per channel | No | No | No | No | Agent roles | **Yes (access/mute/visibility)** |
| Side effects (tasks/observations) | No | No | Custom events | No | Internal notes | **Yes (two output paths)** |
| Provider-agnostic | No (Twilio only) | No (Sinch only) | No (Vonage only) | No (Bird only) | No (Infobip only) | **Yes (Provider ABC)** |
| MCP for AI agents | No | No | No | No | No | **Yes** |
| Content templates | ContentSid references | template_message (channel-specific) | No | hsm type | template type | Not yet |
| Service/tenant isolation | **ConversationService** | **App per project** | No | No | No | **organization_id** |
| Internal/private messages | No | No | No | No | **Internal notes** | **visibility=internal** |

---

## 4. Gaps & Recommendations for Our RFC

### Critical gaps (must address before v1)

| # | Gap | Seen In | Impact | Recommendation |
|---|---|---|---|---|
| 1 | **Content transcoding for non-AI paths** | Sinch | Human sends WhatsApp rich card → SMS user gets nothing | Add `ContentTranscoder` that converts content for target channel capabilities |
| 2 | **Sequential event indexing** | Twilio, Vonage | Better pagination, read tracking, gap detection | Add `index: int` to RoomEvent (sequential per room) |
| 3 | **Read horizon tracking** | Twilio | "Unread count" UIs need this | Add `last_read_index: int` to ChannelBinding or Participant |
| 4 | **Channel fallback** | Sinch, MessageBird | SMS fails → try email | Add `FallbackPolicy` concept (channel priority + timeout) |
| 5 | **Template support** | All (WhatsApp requires it) | Can't use WhatsApp without templates | Add `TemplateContent` to EventContent union |

### Important gaps (should address in early phases)

| # | Gap | Seen In | Impact | Recommendation |
|---|---|---|---|---|
| 6 | **Room timers for auto-transitions** | Twilio | Manual room lifecycle management is error-prone | Add `RoomTimers` (inactive_after, closed_after) |
| 7 | **Correlation ID** | Sinch, Infobip | Hard to link events to external systems | Add `correlation_id: str \| None` to RoomEvent |
| 8 | **Identity merge** | Sinch | Two identities → same person needs explicit merge | Add `fw.merge_identities(source_id, dest_id)` |
| 9 | **Contact/vCard content** | Sinch, Infobip | Share contact information | Add `ContactContent` to EventContent |
| 10 | **List picker content** | Sinch, Infobip | Interactive lists for menus | Add `ListContent` to EventContent or enrich RichContent |

### Nice-to-have (future phases)

| # | Gap | Seen In | Impact | Recommendation |
|---|---|---|---|---|
| 11 | **gRPC callback support** | Sinch | Lower latency for high-volume | Support gRPC as webhook transport |
| 12 | **Async capability query** | Sinch | Query what a contact can receive before sending | Add `fw.query_capabilities(channel_id, address)` |
| 13 | **Message injection** | Sinch | Insert historical messages without triggering hooks | Add `fw.inject_event(room_id, event, skip_hooks=True)` |
| 14 | **Conversation timers** | Twilio | Auto-pause/close after inactivity | Timer system on Room |

---

## 5. Where Our RFC is Already Stronger

| Feature | Our Advantage | CPaaS Comparison |
|---|---|---|
| **AI as a channel** | First-class AI channel with same interface as SMS | No CPaaS has this. They all treat AI as external. |
| **Permission primitives** | access + mute + visibility per channel per room | CPaaS: all participants equal. No per-channel control. |
| **Two output paths** | Events (subject to permissions) + side effects (always flow) | No CPaaS has this concept. |
| **Dynamic permission changes** | mute/unmute/set_visibility at runtime | CPaaS: add or remove participant. No in-between. |
| **Provider abstraction** | Channel type ≠ provider. Swap Twilio for Sinch. | CPaaS ARE the provider. No abstraction needed (but no choice either). |
| **Sync hooks (blocking)** | Hooks can block/modify events in pipeline | Only Twilio has pre-webhooks. Others: post-only. |
| **Hook system depth** | Global + room-level hooks, priority ordering, chaining | Twilio: pre/post URLs only. No chaining or priority. |
| **MCP integration** | AI agents interact via MCP natively | No CPaaS has this. |
| **Channel direction** | inbound/outbound/bidirectional declared on channel | CPaaS: implicit, not declarative. |
| **Media types on channel** | text/audio/video classification | CPaaS: implicit per channel. |
| **Three-level metadata** | Channel.info + ChannelBinding.metadata + EventSource | Twilio comes close with Attributes. Others: limited. |

---

## 6. Design Patterns Worth Adopting

### From Twilio

1. **Sequential message indexing**: `Message.Index` (0, 1, 2...) per conversation. Simple, powerful.
2. **Read horizon**: `lastReadMessageIndex` per participant. Enables unread counts.
3. **Timer-based state transitions**: `TimersInactive` and `TimersClosed` with ISO 8601 durations.
4. **Content Templates (ContentSid)**: Reference pre-built templates by ID. Decouples content creation from sending.
5. **Generous Attributes**: 32KB JSON on every resource. Don't be stingy with metadata limits.

### From Sinch

6. **Channel priority ordering**: `channel_priority_order: ["WHATSAPP", "SMS"]` for fallback.
7. **Auto-transcoding**: Rich content automatically degrades for simpler channels.
8. **Contact merge**: Explicit `contacts:merge` API.
9. **Correlation ID**: First-class field echoed back in all webhooks.
10. **Google-style RPC methods**: `:send`, `:merge`, `:stop` — clear action semantics.

### From Vonage

11. **Event-sourced model**: Everything is an event with a sequential ID. Our timeline is already event-sourced.
12. **Custom event types**: `custom:your_namespace` — extensibility through the event model.
13. **Member state machine**: INVITED → JOINED → LEFT. More granular than just "attached/detached".

### From MessageBird

14. **Automatic channel fallback with timers**: `"fallback": {"from": "+31611111111", "after": "15m"}`.
15. **Channel as first-class resource**: Channels are registered resources with IDs. We already do this.

### From Infobip

16. **Internal notes**: Messages visible only to agents. Our `visibility=internal` covers this.
17. **callbackData echo**: Attach custom data to outbound, echoed in delivery reports.
18. **People CDP integration**: Richer than just identity — tracks behavior, segments, tags.

---

## 7. Recommended RFC Updates (Priority Order)

### Phase 0 additions (model validation)
- Add `index: int` to RoomEvent (auto-incremented per Room)
- Add `last_read_index: int | None` to ChannelBinding
- Add `correlation_id: str | None` to RoomEvent
- Add `RoomTimers` (inactive_after, closed_after with auto-transitions)

### Phase 1 additions (first channels)
- Add `ContentTranscoder` for non-AI cross-channel content conversion
- Add `TemplateContent` to EventContent (required for WhatsApp)
- Add `FallbackPolicy` (channel_priority + timeout)

### Phase 2 additions (dynamic permissions)
- Add `fw.merge_identities(source_id, dest_id)`
- Add `ChannelBinding.state` (invited → joined → left) inspired by Vonage

### Phase 4 additions (more channels)
- Add `ContactContent` (vCard)
- Add `ListContent` (interactive list picker)
- Expand `TemplateContent` for per-channel templates

---

## 8. Architecture Validation

Our RFC's architecture maps well to industry patterns:

```
                    CPaaS Pattern                    Our RFC
                    ─────────────                    ───────
Conversation        Conversation                     Room
                    (channel-agnostic container)     (channel-agnostic container)

Participant         Participant + binding            Channel + ChannelBinding
                    (one channel per participant)    (one channel per binding, with permissions)

Message             Message                          RoomEvent
                    (text/media/template)            (EventContent union + EventSource)

Webhook             Pre/Post webhooks                Sync/Async Hooks (richer)
                    (URL-based, limited)             (code-based, pipeline, priority)

Provider            Platform IS the provider         Provider ABC (swappable)
                    (no abstraction needed)          (Twilio, Sinch, custom)

Identity            Contact/User                     Identity + IdentityResolver
                    (platform-specific)              (pluggable)

AI                  External (not modeled)           AIChannel (first-class)

Permissions         None (all equal)                 access + mute + visibility

Side effects        None                             Tasks + Observations (always flow)
```

**Conclusion**: Our framework takes the best patterns from CPaaS platforms (Room/Conversation abstraction, event timeline, hooks) and adds capabilities no CPaaS provides (AI as channel, permission primitives, provider abstraction, two output paths). The gap analysis above identifies concrete features we should add to be feature-complete for real-world use.
