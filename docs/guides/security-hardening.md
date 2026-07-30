# Security Hardening

RoomKit is a library, not a service: it processes what your application hands
it, and several of its inputs are attacker-controlled by construction. This
page covers the places where that matters and what to do about each.

It is about *deploying* RoomKit safely. To **report a vulnerability**, see
[SECURITY.md](https://github.com/roomkit-live/roomkit/blob/main/SECURITY.md).

## Webhook endpoints

A webhook URL is public by construction: the provider has to reach it, so
anyone else can too. Nothing downstream of your handler can tell a forged
payload from a real one — the parsers produce a perfectly well-formed
`InboundMessage` either way, and `kit.process_webhook()` does not
authenticate. Verifying the signature is the endpoint's job, and it has to
happen before anything else (RFC §17.1).

RoomKit ships the check for the providers that support one. All of them
compare in constant time; Telnyx additionally rejects replays outside a
five-minute window.

| Provider | Method | Header |
|---|---|---|
| Twilio SMS / RCS | `verify_signature(payload, sig, url=...)` | `X-Twilio-Signature` |
| Telnyx SMS / RCS | `verify_signature(payload, sig, timestamp)` | `telnyx-signature-ed25519`, `telnyx-timestamp` |
| Sinch SMS | `verify_signature(payload, sig)` | `X-Sinch-Signature` |
| Messenger | `verify_signature(payload, sig)` | `X-Hub-Signature-256` |
| Telegram | `verify_signature(payload, secret_token)` | `X-Telegram-Bot-Api-Secret-Token` |
| Teams | `process_inbound(payload, auth_header, on_turn)` | `Authorization` (JWT) |

```python
@app.post("/webhooks/sms")
async def inbound(request: Request):
    raw = await request.body()          # raw bytes, before any parsing
    if not provider.verify_signature(
        raw,
        request.headers.get("X-Twilio-Signature", ""),
        url=PUBLIC_WEBHOOK_URL,         # the public URL, not request.url
    ):
        raise HTTPException(status_code=403)

    meta = extract_sms_meta("twilio", dict(await request.form()))
    await kit.process_webhook(meta, channel_id="sms-twilio")
```

Two details are easy to get wrong, and both make every request fail to verify
rather than fail loudly. Twilio signs the **URL** along with the parameters, so
it must be the public URL the provider called — behind a proxy, `request.url`
is not it. And the signature covers the **raw body**: read the bytes before any
framework parses and re-serialises them.

`examples/webhook_signature_verification.py` runs the whole thing offline —
acceptance, a tampered body, a replay aimed at another URL, a missing header.

Providers with no `verify_signature` — WhatsApp, Discord, ElasticEmail,
SendGrid, VoiceMeUp, and the generic HTTP provider, which signs its *outbound*
requests but verifies nothing inbound — leave authentication entirely to your
endpoint. Calling the inherited method on one raises `NotImplementedError`
rather than returning `True`, so it fails closed if you try.

## What `participant_id` is, and is not

Without an `identity_resolver`, a channel stamps the sender id it was given
straight onto the event as `participant_id`, and RoomKit leaves it there. This
is required behaviour, not an oversight — the specification says so explicitly
(RFC §11.6: when resolution is skipped the id "MUST be left as the channel set
it") — because only the channel knows whether its sender ids mean anything.

What follows from it is worth stating plainly, because it is not obvious from
the outside: **`participant_id` is whatever your transport put there.** If your
WebSocket handler takes it from a client-supplied field, then it is
client-controlled, and so is everything the framework decides from it. That
includes authorship: the edit/delete rules (RFC §10.3) establish who may
rewrite or remove a message by comparing `participant_id` to the target
event's. An unauthenticated id there means an unauthenticated author check.

So: derive `sender_id` from your own authenticated session — the token you
validated when the socket opened, not a field in the message — or install an
`identity_resolver` and let the framework resolve it. RoomKit cannot tell the
difference between the two, and does not try to.

Conference participants are the exception, and deliberately so: identity there
is resolved only from `asserted_metadata`, which a backend may populate solely
with values the SFU established (RFC §12.10). Widening that to what a client
supplied at join requires setting `identity_trusts_unasserted_metadata=True`,
which exists to be a visible decision.

## Deploying the SIP backend

`SIPVoiceBackend` is written for a **trusted PBX or SBC in front of it**. Its
port must not be reachable from the open internet. Three properties of the
design follow from that assumption, and each one is a hole if the assumption
does not hold:

- **Authentication is off by default.** `auth_users` and `set_auth_resolver()`
  are both optional, and with neither set every INVITE is accepted. Configure
  one whenever anything other than your own PBX can reach the port.
- **The caller chooses its room.** `X-Room-ID` on the INVITE becomes the room
  the call is routed to, and `X-Session-ID` becomes the session id. Both are
  ordinary SIP headers: whoever sends the INVITE writes them. A PBX that sets
  them itself, and strips whatever the far end sent, is what makes them
  trustworthy — RoomKit cannot tell the difference.
- **The offer chooses where media goes.** The RTP destination comes from the
  SDP. RoomKit rejects addresses that cannot be a destination — `0.0.0.0`,
  port 0, loopback, multicast, link-local — but it cannot reject an address
  that is merely someone else's, because a caller behind NAT legitimately
  advertises an address its packets do not come from. On a reachable port that
  is an amplification primitive: the call becomes an RTP stream aimed at a
  third party.

Three settings bound that last one, and it is worth being precise about which
covers what, because none of them covers all of it:

- `symmetric_rtp=True` follows the address the caller's packets actually come
  from (RFC 4961), so an offer pointing elsewhere stops being followed as soon
  as the caller sends anything of its own. It is also the ordinary fix for
  callers behind NAT. **It does not stop a caller that stays silent**:
  latching only fires on an inbound packet, so an INVITE that advertises a
  third party and then sends nothing keeps the stream aimed there. Requires
  `aiosipua>=0.7.1`; off by default, since it changes how media is addressed
  mid-call.
- `rtp_establishment_timeout` is what bounds the silent case — a session that
  never receives a packet releases its port, so a reflector lasts that long
  rather than forever. On by default (60 s).
- `max_sessions` bounds how many can run at once, answering `503` past the cap.

Authentication is what actually prevents it. The three above limit what an
unauthenticated caller can do; they do not make the port safe to expose.

Protocol traces (`on_trace`, `ON_PROTOCOL_TRACE`) carry SIP messages close to
verbatim. The digest `response` is masked, but everything else — caller,
callee, X-headers, SDP — reaches whatever consumes traces. Treat that stream
as you would the call metadata itself.

## Voice transports that face the internet

`WebTransportBackend` and the WebSocket realtime transport accept sessions
from clients directly, and every accepted session spends STT and TTS on your
account. Both take an `authenticate` callback; `WebTransportBackend` refuses
to start without either that or an explicit `allow_anonymous=True`, because
its default bind is `0.0.0.0` and an open voice endpoint is an open invoice.

## Hook failure semantics

A sync hook that raises, times out, or returns the wrong thing is treated as
**allow** — a broken hook must not take a room down (RFC §9.3). The exceptions
are `BEFORE_TTS` and `ON_TRANSCRIPTION`, whose payload is content a hook may
exist to withhold; those block instead.

The consequence to plan for: **a moderation hook on `BEFORE_BROADCAST` that
crashes lets the content through.** See [Guardrails](guardrails.md) for how to
observe that and how to extend the fail-closed set.

## Secrets and logs

RoomKit does not log message content by default (`ROOMKIT_LOG_CONTENT` gates
it), config objects keep their credentials out of `repr()`, and a source's URL
is stripped of its query string before it reaches a log line or a framework
event — a token in the query string is a common way to authenticate a
WebSocket or SSE endpoint, and `name` travels further than the connection does.

Two things stay verbatim and are worth treating as sensitive: `raw_payload` on
events, and protocol traces when you enable them.
