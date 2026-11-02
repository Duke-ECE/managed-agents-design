# Technical Design

Status: Proposed

## Problem and goals

The console embeds its own HTTP/SSE client. Third-party developers need a supported client for chatting with an existing agent without copying console code. Version 1 should support session creation with an existing `agent_id`, streamed text/events, session continuation, cancellation, and history retrieval.

## Decisions and alternatives

D001: TypeScript/JavaScript is the first SDK language. Package distribution remains open.

D002: each user can generate a named server-side platform API key for a third-party integration. The key acts as that user's identity. It can create and manage only that user's sessions and can create a session only with an existing agent template visible to that user. Key creation/list/revocation requires the user's Supabase login; the key authenticates `/api/sessions*` only. A custom LLM key is a provider credential, not platform authentication. Secrets are shown once, stored as SHA-256 hashes of 256-bit random values, and checked against the store on every request so revocation is immediate. Usage limits remain a future concern.

D003: v1 is text chat. The runtime's default `NullExecutor` returns `tool execution unavailable: sandbox not connected` for read/write/bash/edit. The SDK forwards tool events but neither executes tools nor reports success on their behalf.

## Data model and contracts

The existing API exposes `POST /api/sessions` with `agent_id`, `POST /api/sessions/:id/messages` as SSE, `GET /api/sessions/:id/messages`, `GET /api/sessions/:id/structured`, and `POST /api/sessions/:id/cancel`. Durable chat accepts and echoes `X-Client-Request-Id`. The SDK retains `agent_id`, `session_id`, and the accepted request id. The new `api_keys` table stores owner, name, prefix, hash, creation time, and optional revocation time. No proto change is needed for chat-only v1.

## Execution and recovery

Authenticate, create a session for an accessible existing agent, POST text, parse SSE frames, and continue with the same session id. Distinguish HTTP rejection from an SSE `error` event. Do not silently retry a possibly submitted turn using a fresh request id. Caller cancellation of the HTTP stream must not be presented as guaranteed cancellation of durable execution.

## Verification

- A third-party example creates a session for an accessible existing agent and receives a complete streamed reply.
- A second message continues the session and history can be read back.
- Unauthorized, inaccessible-agent, bad-request, and stream-error cases have documented behavior.
- A dropped connection can be inspected or resumed without creating a duplicate turn.
- Tool behavior matches D003 and does not claim execution when the runtime reports an unavailable executor.

## Open questions

- Which package registry and publication process?
- Should keys receive finer agent or operation scopes in a future version?
- How should tool execution be added in a later version?
- Are SDK consumers server-side only, or must browser CORS be supported?
