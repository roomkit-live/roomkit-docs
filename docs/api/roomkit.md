# RoomKit

::: roomkit.RoomKit
    options:
      show_bases: false
      inherited_members: true
      members:
        - store
        - hook_engine
        - realtime
        - status_bus
        - create_room
        - get_room
        - close_room
        - archive_room
        - check_room_timers
        - check_all_timers
        - update_room_metadata
        - set_agent_response_policy
        - register_channel
        - attach_channel
        - detach_channel
        - mute
        - unmute
        - set_visibility
        - set_access
        - update_binding_metadata
        - get_channel
        - list_channels
        - get_binding
        - list_bindings
        - get_timeline
        - list_tasks
        - delegate
        - cancel_task
        - list_observations
        - join
        - leave
        - process_inbound
        - send_event
        - commit_event
        - deliver
        - start_room_recording
        - room_recordings
        - add_room_recording_track
        - stop_room_recording
        - regenerate_response
        - regenerate_target
        - ensure_participant
        - resolve_participant
        - connect_websocket
        - disconnect_websocket
        - mark_read
        - mark_all_read
        - publish_typing
        - publish_presence
        - publish_read_receipt
        - subscribe_room
        - unsubscribe_room
        - hook
        - "on"
        - identity_hook
        - add_room_hook
        - remove_room_hook

## Exceptions

::: roomkit.RoomNotFoundError

::: roomkit.ChannelNotFoundError

::: roomkit.ChannelNotRegisteredError

## Infrastructure

::: roomkit.core.locks.RoomLockManager

::: roomkit.core.locks.InMemoryLockManager

::: roomkit.voice.auth.AuthCallback

