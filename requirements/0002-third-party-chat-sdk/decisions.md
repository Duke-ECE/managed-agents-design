# Decision Log

## D001: First SDK language

Status: Accepted

Date: 2026-10-29

### Context

The first release needs one supported client language.

### Decision

Build the first SDK for TypeScript/JavaScript.

### Consequences

Other languages can follow after the public chat contract is stable. Package publication remains undecided.

## D002: Third-party authentication

Status: Accepted

Date: 2026-10-29

### Context

The current HTTP API authenticates Supabase users. Third-party applications need their own server-side credentials.

### Decision

Each user can issue separate named platform API keys to third-party integrations. A custom LLM provider key is not a platform credential.

### Consequences

Keys act as their issuing user for session ownership and agent visibility. A key may use `/api/sessions*` only; creating a session requires an existing accessible `agent_id`. The owner can list and revoke keys from the platform. Fine-grained scopes and usage limits are deferred.

## D003: Tool behavior in the first SDK release

Status: Accepted

Date: 2026-11-01

### Context

The runtime currently has a `NullExecutor`; sandbox-backed tool execution is not available to third-party sessions.

### Decision

Version 1 supports text chat and forwards any tool-related stream events as raw events. The SDK does not execute tools or report that a tool succeeded. Tool execution is a separate platform feature.

### Consequences

Integrators must treat tool errors as failures. Adding sandbox or integrator-side tool execution later requires a separate design and verification.

## D004: Release API keys before the durable-session cutover

Status: Accepted

Date: 2026-11-01

### Context

The durable-session requirement still needs cutover and rollback rehearsal. Its backend and console branches contain the API-key changes, but merging those branches would also deploy the unready multi-service cutover.

### Decision

Release the API-key backend and console page on the current v1 session API through isolated branches. Keep the SDK and its durable cancellation and request-deduplication claims unpublished until that contract is deployed and verified.

### Consequences

Server-side callers can issue keys and use the existing create/chat/history session routes. They must inspect history before retrying an uncertain chat submission; the current backend does not deduplicate it. The original draft PRs must be reconciled with this release before they are merged.
