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

## D005: Public integration documentation

Status: Accepted

Date: 2026-11-02

### Context

The console's API guide sits behind login, while application developers need a shareable entry point from the public site.

### Decision

Publish the integration flow and current HTTP/SSE examples on the GitHub Pages documentation site. Link to it from the public frontend landing page. Keep key issuance and agent configuration in the authenticated console. Mark the architecture image as a proposed design rather than the deployed API contract.

### Consequences

The documentation site is the public guide; console key management links to it. Examples must match the deployed backend and must not claim the unpublished SDK or durable retry behavior is available.

## D006: Consistent public-site and console navigation

Status: Accepted

Date: 2026-11-02

### Context

The public site, console, and documentation site have overlapping entry points. Console links currently land on `/agents`, while a direct login lands on `/chat`; the API guide is duplicated inside the key-management page.

### Decision

Keep the product homepage at the frontend origin and the documentation index on GitHub Pages. Add a guarded `/console` entry that resolves to `/agents`, preserve deep links through login, make the public API guide the canonical guide, and keep key issuance on `/keys`. Redirect legacy `/keys?tab=docs` visits to the public guide. Add reciprocal Docs and Console links and a public 404 page.

### Consequences

No origin, OAuth callback URL, or backend CORS change is required. Deploy the frontend route before updating documentation links to `/console`; existing `/agents`, `/chat`, and `/keys` links remain valid.
