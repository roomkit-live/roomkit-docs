# Identity Verification

[Identity resolution](identity-resolution.md) says whose an address is: a
number that resolves to Alice says the phone is Alice's. It does not say that
Alice is the one typing. **Verification** says the person writing has proven
who they are. A service that answers account questions over SMS, WhatsApp or
Telegram needs both.

RoomKit keeps the state and its lifecycle, labels every message, holds an
unverified one from the agents and releases it once verified (RFC §11.7). How a
person proves who they are — a page behind a link, a login, a code replied in
the conversation — stays yours.

Verification applies to an **identified** sender only. Asking an unknown sender
who they are is identification (`IdentityHookResult.challenge()`), not this.

## Quick Start

```python
from roomkit import HookResult, HookTrigger, RoomKit, VerificationPolicy

kit = RoomKit(
    identity_resolver=MyResolver(),
    verification_policy=VerificationPolicy(),  # turns verification on
)


@kit.hook(HookTrigger.ON_VERIFICATION_REQUIRED)
async def ask_to_verify(event, ctx) -> HookResult:
    state = await kit.sender_verification(event, ctx)
    if state is not None and state.request_id is None:  # no link out yet
        request, secret = await kit.request_verification(
            state.identity_id,
            room_id=event.room_id,
            channel_id=event.source.channel_id,
            window_start=event.created_at,
        )
        send_link_later(event.room_id, f"https://bank.example/verify/{request.id}/{secret}")
    return HookResult.allow()  # held until verified
```

Your page checks the person its own way, then tells RoomKit:

```python
from roomkit import VerificationRefusedError, VerificationRequestStatus


async def verify_page(request_id: str, secret: str, pin: str) -> str:
    request = await kit.store.get_verification_request(request_id)
    if request is None or request.status != VerificationRequestStatus.OPEN:
        return "This link is no longer valid."
    try:
        if not pin_matches(request.identity_id, pin):
            counted = await kit.record_failed_attempt(request_id)  # bounded like a wrong secret
            if counted.status == VerificationRequestStatus.FAILED:
                return "Verification failed. Text us again to get a new link."
            return "Wrong PIN."
        await kit.complete_verification(request_id, secret)
    except VerificationRefusedError:
        return "This link is no longer valid."
    return "You're verified."
```

Once verified, what the person wrote while the request was open is released
to the room's agents, which answer it.

## The State

A verification belongs to the **identity**, in an organization: verified once,
the person is verified on every channel and in every room their addresses
reach. `verified_via` names the channel they proved it on, for an integrator
that wants a channel of its own.

| `VerificationStatus` | Meaning |
|---|---|
| `UNVERIFIED` | Never verified, or a request failed |
| `PENDING` | A request is open |
| `VERIFIED` | Proven, until `expires_at` |
| `EXPIRED` | Its time ran out |
| `REVOKED` | Ended by `revoke_verification()` |

`kit.verification_of(identity_id)` reads it. An identity asked again while
verified stays `VERIFIED` until the new request completes or fails.

## Requests

`request_verification(identity_id, room_id=, channel_id=, method=, window_start=)`
opens a request and returns it with its **secret, once**. RoomKit keeps only its
scrypt hash.

| `method` | Secret | Lifetime |
|---|---|---|
| `"link"` (default) | 256 bits, URL-safe: put it in your link | `request_ttl` (600 s) |
| `"code"` | `code_length` digits (6): the person replies with it | `code_request_ttl` (300 s) |

Opening a request cancels the identity's earlier open one: an old link stops
working. A request takes its **room's organization**; a caller passing another
`organization_id` gets `RoomNotFoundError`.

`complete_verification(request_id, secret)` refuses a request that is unknown,
no longer open, expired, or given a wrong secret, with
`VerificationRefusedError.reason`: `not_found`, `not_open`, `expired`,
`mismatch`, `failed`. A mismatch counts as an attempt, and so does your own
check's failure reported with `record_failed_attempt(request_id)`; the last
allowed attempt fails the request. A request that fails while the identity is
verified ends that verification too: whoever could not prove it is not let on
as the person.

!!! warning "The message carrying the secret"
    A message you send through the room (`send_event`) is stored like any
    other: the link or code it carries is readable by whoever reads the
    timeline, an advisor's console included. To keep the secret out of the
    store, send it through the channel's provider directly. A link is a bearer
    secret either way: forwarded, it verifies whoever opens it and passes your
    page's check, so put a check (a PIN, a login) behind it.

## On Every Message

With a `verification_policy`, the inbound pipeline stamps
`event.source.verification` on every message of an identified sender: hooks,
routers, user interfaces and context builders read the same fact. While the
identity is not verified, the message fires `ON_VERIFICATION_REQUIRED` (sync),
whose hooks decide:

| Hook returns | The message |
|---|---|
| `HookResult.allow()` (or no hook) | **Held**: stored and shown to the room's transports (the advisor sees it), read by no agent |
| `HookResult.allow_unverified()` | Passes, labelled: for a general question |
| `HookResult.block(reason)` | Blocked, not stored |

A hook that only watches returns `allow()`: it never lets a message through by
accident. Holding only narrows what a message reached; it never adds a channel.
An edit or a deletion from an unverified sender is refused unless a hook passes
it. `VerificationPolicy(hold_unverified=False)` only labels.

`kit.sender_verification(event, ctx)` gives the hook the sender's verification
in the room's organization: `identity_id`, `status`, and `request_id` when a
request is open. It works on the sender's first message too, before they are a
participant of the room.

!!! note "Hooks run under the room lock"
    Send the link after the hook returns (a background task): `send_event`
    from inside a sync hook waits for the lock the hook is holding.

A sender who writes again while a request is open meets silence unless your
hook answers. Reminding them means opening a new request: a secret is never
kept to be sent again.

## Codes in the Conversation

With `method="code"`, a reply from the request's identity, in its room, whose
whole text is a code of the right length is an attempt: it completes the
request or counts against it, and it is **consumed** — never stored, never
broadcast, read by no agent. Any other message is an ordinary one.

## Release

When a request completes, the held messages its sender wrote while it was open
are released (`release_on_verify=True`): each gets back the visibility it had,
through the store (`ON_EVENT_UPDATED`), and the room's agents answer the last
one as any message of its author, the earlier ones as its context. Nothing is
copied, and a released message is the author's own turn, never an instruction.

Only the request's window is released: from `window_start` (pass the held
message's `created_at`) to the completion, at most `release_max_messages`
within `release_max_age`. What was written before — during a request that
failed, before a verification was revoked — may not be the verified person's,
and stays held. Only what the pipeline held is released: the held mark is the
runtime's, and the pipeline removes one a sender's metadata carries.

## The End

| Hook (async) | Fires when | `event.content.data` |
|---|---|---|
| `ON_VERIFICATION_COMPLETED` | A request completes | `identity_id`, `organization_id`, `request_id`, `channel_id`, `method`, `expires_at` |
| `ON_VERIFICATION_ENDED` | A verification or a request ends | `identity_id`, `organization_id`, `request_id`, `channel_id`, `reason`, `ended` |

`ended` says what ended: `"verification"` when the verification in force ended
(expired, revoked, a request failed while it held), `"request"` when a request
alone did (a link left unanswered, a failure while pending). Both hooks fire in
the room the request was opened in; the end of a verification in the room it
was proven in.

A verification ends **when it expires**, by a timer, not on the person's next
message:

```python
@kit.hook(HookTrigger.ON_VERIFICATION_ENDED, execution=HookExecution.ASYNC)
async def ended(event, ctx) -> None:
    data = event.content.data
    if data["ended"] == "verification" and data["reason"] == "expired":
        await kit.send_event(
            event.room_id, "bank", TextContent(body="Your verification has expired."),
            visibility=data["channel_id"],
        )
```

`kit.revoke_verification(identity_id, reason="advisor")` ends one before its
time and cancels any open request, so a link sent before the revocation never
verifies.

The timers live in the process. `async with RoomKit(...)` (or
`kit.check_verifications()`) ends what expired while nothing ran and re-arms
the rest from the store; `kit.close()` stops them.

## Policy

| `VerificationPolicy` field | Default | Effect |
|---|---|---|
| `ttl` | 900 | Seconds a verification stays in force |
| `request_ttl` | 600 | Seconds a link request may be completed in |
| `code_request_ttl` | 300 | Seconds a code request may be completed in |
| `max_attempts` | 3 | Wrong secrets or failed checks before a request fails |
| `code_length` | 6 | Digits of a code (6 to 12) |
| `hold_unverified` | `True` | Hold an unverified sender's message; `False` only labels |
| `release_on_verify` | `True` | Release held messages once verified |
| `release_max_messages` | 20 | At most this many are released, the latest |
| `release_max_age` | 3600 | Older held messages are not released |

## Organizations

Verification state is scoped to the organization, as identities are. A message
is labelled by the verification of its **room's** organization: one proven to
organization A never verifies the person in organization B's conversation.

## Storage

The in-memory, SQLite (schema v5, migrated in place) and PostgreSQL stores keep
verifications and requests. A store of your own keeps working without them: the
operations refuse by name until you implement them.

## Example

[`examples/identity_verification.py`](https://github.com/roomkit-live/roomkit/blob/main/examples/identity_verification.py)
runs with mock providers: a held question answered once Alice passes the PIN
page, a code replied in the conversation, and an advisor ending a verification.

## See Also

- [Identity Resolution](identity-resolution.md) — who an address is
- RFC §11.7 — the normative rules
