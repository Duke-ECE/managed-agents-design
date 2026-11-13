# Decision Log

Append decisions for this requirement using the entry structure below.
Use local IDs such as D001 and D002. Proposed entries may be revised.
When replacing an accepted decision, append a new entry, mark the old entry
Superseded, and link both entries. Editorial corrections may be made in place.

## D001: Freeze thinking mode at session creation

Status: Accepted

Date: 2026-11-13

### Context

Users need to select a mode without passing it on every chat call. Existing
session configuration is frozen at admission; message-level changes would
make restart and replay behavior ambiguous.

### Decision

Resolve session override > template setting > compatibility default once,
persist the effective value before the first turn, and reject mode changes
on message requests. A new mode requires a new session.

### Consequences

CLI continuation requires only the session id. Existing sessions remain
stable when a template changes. A template provides a reusable preference.

### Related work

[Frozen configuration](../0001-durable-context-compaction/design.md).

## D002: Normalize the public mode and limit initial provider support

Status: Accepted

Date: 2026-11-13

### Context

DeepSeek uses provider-specific `thinking` and `reasoning_effort` fields.
Silently ignoring an unsupported explicit mode would mislead API clients.

### Decision

Expose `thinking_mode` with `off/low/high/max/provider_default`. Map explicit
modes for official DeepSeek Flash only; reject unsupported combinations before
creating a session. Keep DeepSeek Flash's current `off` default when omitted.

### Consequences

The public API is independent of DeepSeek's wire shape. Adding another
provider later requires a capability mapping and tests.

## D003: Preserve provider reasoning across tool-call recovery

Status: Accepted

Date: 2026-11-13

### Context

DeepSeek requires all previous `reasoning_content` to be passed back when a
request carries `tools`. The live pi agent retains it, but the v1 transcript
currently stores only aggregated assistant text and tool events. Hydration
after a runtime restart therefore loses reasoning content and may produce a
provider 400. The owner transcript API must not expose private reasoning.

### Decision

Keep thinking-enabled releases in draft until a durable, server-private
reasoning representation is written and hydrated in message order, with the
owner transcript redacting it. Verify continuation after a runtime restart
with a tool call and a later user turn against a fake provider that checks the
outgoing request. Do not rely on an empty `reasoning_content` placeholder.

### Consequences

The first production rollout needs coordinated runtime and session-manager
changes beyond the public mode field. This also protects history quality
after a restart.
