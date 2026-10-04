# Built-in Channels

`ConferenceChannel` has a page of its own: [Conference](conference.md).

::: roomkit.SMSChannel

::: roomkit.RCSChannel

::: roomkit.EmailChannel

::: roomkit.AIChannel

## Turn notes

What changes from one turn to the next rides the turn's input as notes (RFC §6.4).
A `BEFORE_AI_GENERATION` hook adds to them; a debug view splits them off.

::: roomkit.add_turn_note

::: roomkit.split_turn_notes

::: roomkit.TURN_NOTES_HEADER

## Tool Search

The discovery tools a channel serves itself under Tool Search.

::: roomkit.TOOL_SEARCH_INFRA_TOOL_NAMES

::: roomkit.TOOL_FIND_TOOLS

::: roomkit.TOOL_LIST_TOOLS

::: roomkit.ACPChannel

::: roomkit.WebSocketChannel

::: roomkit.VoiceChannel

::: roomkit.RealtimeVoiceChannel

::: roomkit.WhatsAppChannel

::: roomkit.MessengerChannel

::: roomkit.TeamsChannel

::: roomkit.HTTPChannel

::: roomkit.TelegramChannel

::: roomkit.WhatsAppPersonalChannel

::: roomkit.TransportChannel

## WebSocket Streaming

::: roomkit.channels.websocket.StreamStart

::: roomkit.channels.websocket.StreamChunk

::: roomkit.channels.websocket.StreamEnd

::: roomkit.channels.websocket.StreamMessage

::: roomkit.channels.websocket.StreamSendFn
