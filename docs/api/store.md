# Store

Keyed proactive voice delivery uses atomic voice reservations in addition to
event storage. Built-in stores implement claim and conditional completion;
SQLite and Postgres preserve these receipts across restart until room deletion.
Custom stores must implement the same contract before accepting keyed voice
injection. See [voice idempotency](../guides/delivery.md#voice-idempotency-and-uncertainty).

::: roomkit.store.base.ConversationStore

::: roomkit.store.memory.InMemoryStore

::: roomkit.store.sqlite.SQLiteStore

::: roomkit.store.sqlite.SQLiteSchemaError

::: roomkit.store.postgres.PostgresStore
