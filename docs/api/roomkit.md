# RoomKit

::: roomkit.RoomKit
    options:
      show_bases: false
      members:
        - store
        - hook_engine
        - realtime
        - create_room
        - get_room
        - close_room
        - check_room_timers
        - check_all_timers
        - update_room_metadata
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
        - list_observations
        - process_inbound
        - send_event
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
