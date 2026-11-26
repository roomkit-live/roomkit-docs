# Telemetry

Observability and tracing for RoomKit applications. See the [Telemetry guide](../guides/telemetry.md) for usage examples.

## Core

::: roomkit.telemetry.TelemetryProvider

::: roomkit.telemetry.TelemetryConfig

::: roomkit.telemetry.Span

::: roomkit.telemetry.Attr

::: roomkit.telemetry.SpanKind

## The current span

A provider written outside RoomKit reads the span the current work runs
under, and the context its provider gave that span, to start its own spans
in the same trace.

::: roomkit.telemetry.get_current_span

::: roomkit.telemetry.get_current_telemetry_ctx

## Built-in Providers

::: roomkit.telemetry.ConsoleTelemetryProvider

::: roomkit.telemetry.OpenTelemetryProvider

::: roomkit.telemetry.NoopTelemetryProvider

::: roomkit.telemetry.MockTelemetryProvider
