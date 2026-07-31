# Conference

Multi-party conferences on an external SFU: RoomKit orchestrates and joins as
a bot participant; the SFU owns the media plane (RFC §12.10). See the
[Conference guide](../guides/conference.md) for concepts and wiring.

## Channel

::: roomkit.ConferenceChannel

## Backend

::: roomkit.ConferenceBackend

### LiveKit

Install with: `pip install roomkit[livekit]`

::: roomkit.LiveKitConferenceBackend

::: roomkit.LiveKitConfig

### Mock

The test double, with fault injection — see
[Testing Patterns](../guides/testing-patterns.md#mock-conference-backend).

::: roomkit.MockConferenceBackend

::: roomkit.MockTrackFormat

::: roomkit.MockUtterance

::: roomkit.MockDelivery

::: roomkit.MockFaults

## Access and grants

::: roomkit.ConferenceAccess

::: roomkit.ConferenceGrants

::: roomkit.ConferenceCapability

## Media models

::: roomkit.ConferenceParticipant

::: roomkit.ConferenceTrack

::: roomkit.TrackKind

::: roomkit.BotSession

## Pipeline payloads

::: roomkit.ConferenceTranscription

::: roomkit.ConferenceBargeIn

## Interruption

::: roomkit.ConferenceInterruptionConfig

::: roomkit.ConferenceInterruptionScope

## Recording

::: roomkit.ConferenceRecordingConfig

::: roomkit.ConferenceRecordingMode

::: roomkit.ConferenceRecordingStarted

::: roomkit.ConferenceRecordingStopped

## Errors

`ConferenceAlreadyAttachedError`, `ConferenceCapabilityError`,
`ConferenceCloseError` and `ParticipantNotAdmittedError` are documented on the
[Errors page](errors.md).
