# Delivery Plan

Status: In progress.

## Implementation sequence

| Order | Repository | Deliverables |
| --- | --- | --- |
| 1 | protos | Versioned canonical-message, execution, and checkpoint contracts; generated code; release tag |
| 2 | session-manager | Timestamped schema migrations, transactional PostgREST operations, authorization, request state, leases |
| 3 | agent-runtime | Canonical messages, incremental writes, deterministic hydration, tool recovery, compaction |
| 4 | managed-agents-backend | Request identity, versioned APIs, immutable templates and archival |
| 5 | managed-agents-frontend | Structured message rendering, request status, Clone and Archive flows |
| 6 | All affected services | Cross-service verification and coordinated cutover |

Implementation may overlap after contracts are fixed; schema and server support
must precede enabling clients.

## Executable checklist

An item is checked only after its implementation is committed and the listed
verification has passed. PR creation alone does not complete an item.

### Phase 0: Contract and delivery control

- [x] Mark the requirement Implementing and establish this evidence checklist.
- [x] Add `session.v2` canonical-message, request-state, lease, mutation, and
  checkpoint contracts.
- [x] Add `runtime.v2` durable request, cancellation, and structured stream
  contracts.
- [x] Run proto lint, breaking-change detection, generation, and Go build.
- [x] Merge the proto PR, publish a semver tag, and record the immutable tag here.

Contract tag: [`v0.8.0`](https://github.com/Duke-ECE/protos/releases/tag/v0.8.0)
(`f3db971`), additive over `v0.7.0` — new `session/v2` and `runtime/v2`
packages, no change to the v1 packages.

Exit criterion: consumers can pin a reviewed v2 contract tag. Met: consumers pin
`v0.8.0`. Until the tag existed, consumer branches could test against a temporary
local module replacement, which must never be committed.

### Phase 1: Durable storage foundation

- [x] Add a non-destructive migration for canonical message fields, request-root
  state, session revisions, fenced leases, immutable session configuration,
  checkpoints, and idempotency receipts.
- [x] Add and validate indexes for latest-window history, request history,
  request deduplication, and one nonterminal request per session.
- [x] Add short Postgres transaction functions for request admission, lease
  operations, incremental append, terminal completion, cancellation, and
  checkpoint publication.
- [x] Verify migrations from both an empty database and the existing v1 schema
  with retained rows.
- [x] Verify RLS remains enabled with no anon/authenticated policies on every
  service-owned table.

Exit criterion: all durable state transitions are atomic inside Postgres and can
be invoked through PostgREST without a read-max-write sequence race. Met: every
transition runs in one transaction function; `scripts/verify-db.sh` exercises the
constraints, indexes, locks, fencing, revision races, RLS, and function
privileges on real Postgres.

### Phase 2: session-manager v2

- [x] Add v2 domain types and validation without importing transport or storage.
- [x] Add a behavior-parity in-memory store for the expanded persistence port.
- [x] Implement PostgREST adapters for transactional RPCs and cursor reads.
- [x] Register `session.v2.SessionService` alongside v1 during migration.
- [x] Enforce owner-versus-service-token authorization and credential redaction.
- [x] Cover deduplication, hash conflicts, request state transitions, revision
  conflicts, lease fencing/expiry, immutable messages, safe checkpoints,
  pagination, end/delete races, and cancellation with unit tests.
- [x] Pass `scripts/check.sh`, gofmt, build, vet, unit tests, and whole-service
  integration tests using Go 1.25 with `GOTOOLCHAIN=local`.

Exit criterion: session-manager is a complete privilege and persistence boundary
for the v2 state machine, while v1 remains usable for controlled migration. Met:
both contract versions are registered over one store; the v2 rules, the
transaction functions, and the adapter are covered at unit, adapter, transport,
and whole-service levels.

Deviations and open items:

- Between-request compaction publishes under the next request's lease: the
  checkpoint's `active_request_message_id` is the new request while
  `covered_through_seq` sits at the last completed-request boundary. Publication
  while a session is idle is deliberately unsupported (a lease belongs to a
  request).
- `GetExecutionContext` returns session, frozen config, and active checkpoint
  only; retained messages are read through `GetMessages` windowed pagination so a
  truncated hydration can never be mistaken for a complete one.
- Legacy v1 `system`/`config` turns have no `session.v2.MessageRole`; the
  transport maps them to `MESSAGE_ROLE_UNSPECIFIED`. Whether to import existing
  development histories or reset remains an open delivery item (see below).

### Phase 3: agent-runtime durable execution

Implemented on the `docs/link-compaction-design` branch behind
[Duke-ECE/agent-runtime#1](https://github.com/Duke-ECE/agent-runtime/pull/1);
items whose listed verification has actually passed are checked, and the rest
record exactly what is still missing.

- [x] Pin the released proto tag and sync vendored proto files through the
  repository script.
- [x] Replace turn-end fire-and-forget persistence with request admission and
  acknowledged incremental durable boundaries. Verified end to end: admission
  precedes lease acquisition, the assistant tool-call message is acknowledged
  before dispatch, the observed result is acknowledged before the next model
  call linked by the original tool-call id, and exactly one terminal write
  carries the final output.
- [x] Acquire, renew, and fence a 90-second execution lease; stop writes and
  inference immediately after lease loss. Acquisition and use are verified end to
  end; renewal, expiry takeover, stale-generation rejection, and the
  stop-inference-on-lease-loss abort are unit-tested in `DurableExecution`.
- [x] Preserve ordered text/tool blocks, stable tool-call IDs, result links,
  provider continuation metadata, and one usage record per model call. Covered by
  the canonical model, the client wire round trip, and the handler's per-model-call
  usage.
- [x] Assemble model context deterministically from frozen config, active
  checkpoint, and retained canonical messages. Implemented with backwards
  pagination that stops at the checkpoint cutoff.
- [x] Add a model-aware token estimator and check the input budget before every
  model call, including tool-loop calls. `prepareNextTurnWithContext` checks before
  each model call; the estimator is unit-tested.
- [ ] Implement safe-cut compaction, summary generation, compare-and-publish,
  mid-request continuation, and summary timeout/failure behavior. Implemented and
  unit-tested (`src/compaction.ts`) and wired into `prepareNextTurnWithContext`,
  which hands the compacted context back for the same request; no handler-level
  test yet.
- [x] Reconcile restart/cancellation behavior without replaying completed tools.
  Cancellation and reconnect-dedup are implemented, and a tool call left without
  a persisted result by a crash is now detected on hydration
  (`unresolvedToolCallsInHistory`) and excluded from model context rather than
  replayed as a dangling call — replaying it would violate the provider protocol
  and invite a fabricated result. The mirror case, an orphan result whose call
  was summarized away, is dropped too, and the unknown outcome is reported as
  telemetry so the turn can require an explicit retry decision. Reconciling
  through an executor's own status/idempotency remains future work, because no
  executor is wired yet.
- [x] Cover crash boundaries, duplicate delivery, stale leases, compaction races,
  long tool output, missing provider usage, and cache invalidation in tests.
  Handler-level coverage exists for a completed turn, a duplicate delivery
  (reconnect), a failed model call, and a tool turn exercising both incremental
  boundaries. Unit coverage exists for stale leases, lease expiry, compaction
  failures, long tool output (the model-facing cap and its named reference),
  missing provider usage (a zero or absent report must not set an estimate
  baseline), and cache invalidation (any model-facing change invalidates the
  estimator's baseline). Crash recovery is covered across a restart with a
  stateful session-manager fake: a crash after admission reconnects without
  re-admitting, a crash after the tool result keeps the pair and replays no
  tool, and a crash after the terminal write cannot extend the sealed
  transcript. Compaction races are covered at the publication level (parent,
  config-hash, revision, and lease guards) but not against a concurrent
  publisher.
- [x] Pass TypeScript strict build and all Node tests. `npm test` green (148
  tests) under `tsc --strict`.

Telemetry: `src/telemetry.ts` records compactions (token reduction derived from
before/after, duration, checkpoint id, covered-through sequence, source
revision, summarizer model, prompt version), lease conflicts, acknowledged write
latency and message counts with their outcome (a refused write is measured too,
so a slow failure is distinguishable from a slow success), and hydration
failures — one JSON object per line, built from explicit field lists so no path
can attach message content, a prompt, or a credential. Wired into compaction,
lease loss, and both durable writes.

Partial evidence: `src/canonical.ts`, `src/tokens.ts`, `src/durable-client.ts`,
`src/durable-execution.ts`, `src/compaction.ts`, `src/runtime-events.ts`, and
`src/durable-runtime.ts` (the `runtime.v2.AgentService` handler, registered
alongside v1 and failing closed without a session-manager), with matching tests.
`npm test` green (148 tests) at `3ad7391`, including the handler-level end-to-end
tests and crash recovery across a restart.

Known gaps carried forward: crash-time reconciliation of a tool call whose
outcome is unknown is not implemented; a first attempt at the handler was
discarded rather than committed after review found a `require` in ESM, an
undeclared live-session field, and an empty compaction message list; and the
frozen configuration's `credential_ref` is not resolvable because private
credential storage does not exist yet, so the process `LLM_API_KEY` supplies the
secret in the meantime.

Exit criterion: a request can survive process loss at every durable boundary and
long contexts compact without overwriting source messages.

### Phase 4: backend admission and immutable templates

In progress on `feat/agent-template-archive` behind
[Duke-ECE/managed-agents-backend#1](https://github.com/Duke-ECE/managed-agents-backend/pull/1).
The template lifecycle slice is done; v2 orchestration, request identity,
structured history, and archived-template admission are not.

- [ ] Pin the released proto tag and switch session/runtime orchestration to v2.
  The tag is pinned (`v0.8.0`); `session.v2` serves canonical history reads and
  canonical admission, and the chat path now runs on `runtime.v2` — all behind
  `CANONICAL_SESSIONS`, the single cutover switch. Durable turns stream over SSE
  under the same event names as v1. What remains is the frontend's structured
  rendering (the v2 `done` frame carries `aggregate_usage` rather than v1's flat
  token counts) and rehearsing the cutover itself.
- [ ] Generate or accept a stable client request ID and deterministic request hash
  for every chat submission and reconnect. Wired into the durable chat turn:
  `Service.ChatTurn` derives the identity and passes it to `runtime.v2`, whose
  request root carries `client_request_id` and `request_hash`, so session-manager
  deduplicates a reconnect and refuses the same id carrying different content.
  Content is encoded with the canonical binary encoding (protojson is not stable
  across releases) with each block length-prefixed, so an identical replay hashes
  identically and a changed one does not. The route is live: a durable turn
  streams over SSE and echoes the identity in `X-Client-Request-Id`, so a browser
  that dropped a connection can replay it and be deduplicated.
- [x] Expose structured message history and request execution state over HTTP.
  `GET /api/sessions/:id/structured` serves the canonical session.v2 history:
  ordered content blocks with tool-call/result links, per-message completeness,
  and the request execution state on the initiating root, paginated like the v1
  transcript. The session slice gained a `CanonicalStore` port, attached through
  a functional option so the existing call sites were untouched, and a missing
  v2 client answers not-implemented rather than an empty history.
- [ ] Replace template mutation/deletion with create, clone, and archive; reject
  new sessions from archived templates while preserving existing sessions.
  Clone and archive are implemented end to end (migration, domain rules, store
  port, PostgREST adapter, HTTP routes, tests) per decision D001: templates are
  immutable, cloning copies configuration into a new private identity, and
  archiving is owner-only, idempotent, and content-preserving. Admission
  revalidates archival server-side: a new session from an archived template is
  refused before the runtime or session-manager is touched, while resume is
  deliberately not gated so existing sessions keep running, and the template
  stays readable and cloneable. Editing is now genuinely impossible rather than
  merely discouraged: PATCH returns a uniform 400 (`ErrImmutableMutation`) that
  is produced before any ownership lookup, so it cannot probe another user's
  templates, and the mutation path was removed from the domain, the store port,
  the adapter, and the transport rather than left unreachable. `DELETE` and the
  frontend's Edit action remain; both are the coordinated API+UI half.
- [ ] Resolve and persist one frozen session configuration at admission, using
  credential references rather than transcript secrets. Implemented behind
  `CANONICAL_SESSIONS` (default off): one session.v2 call writes the session
  record and its frozen configuration together, so the resolved model, system
  prompt, tools, and limits are fixed from the first turn. No API key reaches
  the durable record — the credential is an (empty, for now) reference — and the
  config hash binds a checkpoint to the configuration it was built under. The
  flag keeps admission on v1 until the canonical schema is applied and the
  runtime switches to v2, which is the coordinated cutover; canonical reads are
  already always-on. Two known gaps: model limits are the same constants the
  runtime's adapter uses today rather than per-model limits, and
  `credential_ref` is empty because private credential storage does not exist.
- [ ] Map revision/state/lease conflicts to stable HTTP/SSE errors and preserve
  cancellation as an allowed operation while a session is busy. The HTTP half is
  done: ABORTED (lost, expired, or contended lease; revision and mutation
  conflicts) maps to 409 and INVALID_ARGUMENT to 400, where both previously
  surfaced as 500, and the mapping is pinned by a table test. UNAUTHENTICATED
  deliberately stays 500 so an upstream service-token misconfiguration cannot
  tell a browser to re-authenticate. The SSE half is done too: a failed chat
  stream now reports whether it is worth retrying, derived from the gRPC code
  (UNAVAILABLE, ABORTED, DEADLINE_EXCEEDED, RESOURCE_EXHAUSTED are retryable)
  instead of the hardcoded false that made every transient failure look
  permanent. The busy-session cancellation path waits for the v2 chat
  orchestration.
- [x] Pass gofmt, build, vet, unit tests, and whole-service integration tests.
  `gofmt -l .` clean, `go build ./...`, `go vet ./...`, `go test ./...`, and
  `./scripts/check.sh` green at the template-lifecycle commits (`5b6d2bb`,
  `323143b`).

Exit criterion: browser requests are idempotent, session admission is race-safe,
and templates/configuration cannot silently mutate a running session. Partially
met: templates can no longer mutate a session through the new lifecycle, but
admission is not yet race-safe because session creation still runs on v1.

### Phase 5: frontend structured history

- [x] Render canonical ordered blocks with tool-call/result associations. The
  live view normalizes both contracts at the boundary: a durable `tool_result`
  (ordered content blocks plus a status enum) and a v1 one (a flat
  ok/output/error triple) both become the shape the renderer knows, so a turn
  renders identically before and after the cutover. Text blocks are flattened in
  order, and a failure shows its error_code when it has one and otherwise its
  content, so a tool that failed *with* output still shows that output. The
  durable `done` frame is normalized the same way (its per-model-call
  `aggregate_usage` becomes one total for the turn).
- [x] Show queued, running, completed, failed, cancelled, interrupted, partial,
  and provisional states without presenting unsaved output as durable. Streamed
  text is marked provisional until the done frame, and an interrupted turn is
  marked "partial — not saved" with a resend. Request-level state comes from the
  durable record: on opening a session the console reads the latest root's
  execution status from the canonical history and says, in plain terms, that the
  request is queued, still running on the server, failed, cancelled, or
  interrupted. A completed request says nothing (the transcript shows it) and the
  notice is suppressed while a live turn streams, since the stream is then the
  better source.
- [x] Generate and reuse stable request IDs across SSE reconnects. The client
  generates an id per submission, sends it as `X-Client-Request-Id`, and records
  the id the backend confirmed on the turn. The partial-turn marker offers a
  resend that re-drives the same bubble under that identity, so session-manager
  recognises the request and the runtime answers with its existing state instead
  of running the turn twice. A deduplicated resend returns no new output, so the
  bubble says what happened — the turn was accepted, it is not a failure, and the
  transcript already holds it — rather than showing an empty reply.
- [x] Replace Edit/Delete template actions with Clone/Archive and filter archived
  templates from the default new-session picker. Delete became Archive (a
  non-destructive confirm that keeps the row in place and shows an Archived
  badge), Edit is gone entirely — the drawer's edit state, its PATCH branch, and
  the `updateAgent` client were removed rather than hidden, so the type system
  enforces that a template can never be edited — and the new-chat picker derives
  a selectable list that excludes archived templates while still resolving an
  open session's `agent_id` to a display name. Own templates offer Clone and
  Archive; platform and archived ones offer View and Clone.
- [ ] Preserve cursor pagination, cache revalidation, cancellation, and history
  navigation under the v2 response schema.
- [ ] Pass the TypeScript and Vite production build.

Exit criterion: UI state is a faithful projection of canonical storage and does
not imply that provisional deltas or interrupted output were committed.

### Phase 6: integration, cutover, and rollback

- [ ] Run a multi-service local integration covering create, chat, tool loop,
  resume, cancel, end, delete, and template archive/clone.
- [ ] Run multi-replica failure tests for runtime death, lease expiry/takeover,
  duplicate request delivery, and stale mutation rejection.
- [ ] Run long-session tests that trigger between-request and mid-request
  compaction, then verify deterministic hydration after restart.
- [ ] Verify owner isolation, service-token paths, RLS denial, credential
  non-disclosure, and log/fixture secret scanning.
- [ ] Define the development-data migration decision and rehearse rollback with
  checkpoint-aware binaries before deployment.
- [ ] Record every merged PR, pinned version, deployment, smoke test, deviation,
  and rollback artifact below.

Exit criterion: all acceptance criteria have reproducible evidence and the
requirement can be marked Completed without relying on uncommitted local state.

## Open implementation details

- Final protobuf and HTTP/SSE schemas.
- Credential storage and key-management migration.
- Coordination of template archival and session-creation admission.
- Provider adapter usage normalization and continuation formats.
- Tool execution reconciliation and fencing capabilities.
- Whether to import existing development histories or perform an explicitly approved reset.

## Verification

### Recorded evidence

| Date | Repository | Revision | Verification |
| --- | --- | --- | --- |
| 2026-09-19 | protos | `8202acb` | `buf lint`; `buf breaking --against '.git#branch=main,subdir=proto'`; `buf generate`; `GOTOOLCHAIN=local go build ./...` |
| 2026-09-19 | protos | `f3db971` | PR CI green (lint, breaking, generation drift, `go build`); `main` fast-forwarded; `v0.8.0` tagged and pushed |
| 2026-09-19 | session-manager | `70d87f7` | Migrations applied to Postgres 16 from an empty database and from the v1 schema with retained rows; `supabase/tests/durable_context.sql` asserts constraints, indexes, locks, lease fencing, revision races, RLS, and function privileges via `scripts/verify-db.sh` |
| 2026-09-19 | session-manager | `07d5209` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...`; `go test -race ./...`; `./scripts/check.sh`; `DATABASE_URL=... ./scripts/verify-db.sh` (fresh + upgraded) |
| 2026-09-19 | agent-runtime | `5961397` | `npx tsc --noEmit`; `npm test` green (90 tests) — vendored v0.8.0 v2 contracts, canonical message model, token estimator |
| 2026-09-19 | agent-runtime | `5554602` | `npm test` green (103 tests) — session.v2 durable client (real gRPC round trip over the vendored contract) and the durable request lifecycle; PR CI green |
| 2026-09-19 | agent-runtime | `1c0f55b` | `npm test` green (114 tests) — safe-cut compaction, summary deadline/retry/failure, compare-and-publish |
| 2026-09-19 | agent-runtime | `842d520` | `npm test` green (123 tests) — pure agent-event → runtime.v2 stream translation |
| 2026-09-19 | agent-runtime | `5b23f77` | `npm test` green (125 tests) — `runtime.v2.AgentService` handler registered alongside v1; PR CI green |
| 2026-09-19 | agent-runtime | `9570606` | `npm test` green (126 tests) — handler end-to-end tests (completed turn, reconnect dedup, failed model call) over the real pi loop with a scripted stream |
| 2026-09-19 | managed-agents-backend | `5b6d2bb` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...`; `./scripts/check.sh` — clone/archive rules, adapter, HTTP routes, and error mapping covered |
| 2026-09-19 | managed-agents-frontend | `e1f01fc` | `npm run build` green — durable done frame consumed (per-call aggregate normalized to a turn total) |
| 2026-09-19 | managed-agents-frontend | `85bf519` | `npm run build` green — durable request state read on session open and shown in plain terms; absent when session.v2 is unavailable |
| 2026-09-19 | managed-agents-frontend | `3b8aff5` | `npm run build` green — resend re-drives an interrupted turn under its original identity and explains a deduplicated replay instead of showing an empty reply |
| 2026-09-19 | managed-agents-frontend | `f9c355a` | `npm run build` green — request identity generated, sent, and stored per turn (replay affordance deliberately not shipped; see the item note) |
| 2026-09-19 | managed-agents-frontend | `5adf617` | `npm run build` green — tool results rendered from either contract via a boundary normalizer |
| 2026-09-19 | managed-agents-frontend | `27e37c9` | `npm run build` (`tsc -b && vite build`) green — Edit state removed (type-enforced), own templates offer Clone/Archive only |
| 2026-09-19 | managed-agents-frontend | `db1b5fe` | `npm run build` (`tsc -b && vite build`) green — archived templates excluded from new-chat selection, still resolved for display |
| 2026-09-19 | managed-agents-frontend | `a30be68` | `npm run build` (`tsc -b && vite build`) green — archive replaces delete on the Agents page, archived state surfaced |
| 2026-09-19 | managed-agents-backend | `2f42e16` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...` (10 packages); `./scripts/check.sh` — durable chat turn with a content-bound request identity, replay stability and changed-content divergence covered |
| 2026-09-19 | managed-agents-backend | `2311436` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...` (10 packages); `./scripts/check.sh` — frozen-config admission behind a cutover flag, credential never persisted, config hash bound by a field-mutation test |
| 2026-09-19 | managed-agents-backend | `a73183b` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...` (10 packages); `./scripts/check.sh` — SSE stream retryability derived from the gRPC code, pinned by a table test |
| 2026-09-19 | managed-agents-backend | `f698682` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...` (10 packages); `./scripts/check.sh` — ABORTED/INVALID_ARGUMENT mapping pinned by a table test |
| 2026-09-19 | managed-agents-backend | `e40ef01` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...` (10 packages); `./scripts/check.sh` — structured history route, ownership pass-through, not-implemented and bad-window paths covered; protos pinned to v0.8.0 |
| 2026-09-19 | managed-agents-backend | `1e75113` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...`; `./scripts/check.sh` — PATCH refused uniformly, mutation path removed, row verified untouched |
| 2026-09-19 | managed-agents-backend | `bdfc8d8` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...`; `./scripts/check.sh` — request identity generation, validation, and deterministic hashing; the session slice's first unit tests |
| 2026-09-19 | managed-agents-backend | `323143b` | Same gates green — archived-template admission refused (410) before any runtime or session-manager call, with archived reads still available |
| 2026-09-19 | agent-runtime | `3ad7391` | `npm test` green (148 tests) — crash recovery across a restart (admission, mid-turn, post-terminal) with a stateful session-manager fake; missing provider usage |
| 2026-09-19 | agent-runtime | `3fb5858` | `npm test` green (144 tests) — crash-dangling tool calls detected, excluded from context, and reported |
| 2026-09-19 | agent-runtime | `a9b6f56` | `npm test` green (138 tests) — model-facing tool-output cap with a named reference to the preserved original |
| 2026-09-19 | agent-runtime | `045cb90` | `npm test` green (133 tests) — durable-write latency and outcome emitted from a finally block, so refused writes are measured |
| 2026-09-19 | agent-runtime | `d5bfcc2` | `npm test` green (131 tests) — structured telemetry with a credential/body-free record shape |
| 2026-09-19 | agent-runtime | `c8580ec` | `npm test` green (127 tests) — tool-boundary persistence verified through the handler (assistant call before dispatch, result before the next model call, link by original id) |

| 2026-09-19 | session-manager | `6385031` | Non-destructive canonical schema: message fields, request-root state, revisions, lease columns, immutable config, checkpoints, mutation receipts; legacy v1 rows preserved as format 0 with last_seq backfilled (proved by scripts/verify-db.sh on a fresh database and on a v1 upgrade with retained rows) |
| 2026-09-19 | session-manager | `ff095a3` | Transaction functions for admission/dedup, lease acquire/renew/release, append, finish, cancel, and checkpoint publication; failures carry a machine-readable reason in SQLSTATE DETAIL, execute is granted to service_role only (asserted by supabase/tests/durable_context.sql) |
| 2026-09-19 | session-manager | `6149d01` | Durable domain types, the DurableStore port, and the v2 business rules, with no transport or storage imports |
| 2026-09-19 | session-manager | `dde37e6` | PostgREST adapter over the transaction functions plus the behaviour-parity in-memory store |
| 2026-09-19 | session-manager | `1239ccc` | session.v2 gRPC transport registered alongside v1; main assembles both services over one store |
| 2026-09-19 | session-manager | `af5e56e` | AGENTS.md updated for the v2 contract, the transaction functions, and the database verification script |
| 2026-09-19 | agent-runtime | `87eb529` | Pre-existing branch commit linking the shared design; the basis this work builds on |
| 2026-09-19 | agent-runtime | `28b3c03` | Vendored the released v0.8.0 session/v2 and runtime/v2 contracts through scripts/sync-proto.sh |
| 2026-09-19 | agent-runtime | `ca1c335` | Awaited session.v2 client with wire conversion and aborted-vs-conflict classification; the round trip over the vendored contract caught a real bug (the draft completeness field is status, not message_status) |
| 2026-09-19 | agent-runtime | `5303b2d` | Exported the wire block decoder the v2 handler boundary needs |
| 2026-09-19 | managed-agents-backend | `e0af002` | AGENTS.md documents the clone/archive lifecycle, owner-only and platform-read-only rules, and the admission revalidation (including that resume deliberately does not re-check) |
| 2026-09-19 | managed-agents-backend | `d4c5b17` | Durable chat streamed over SSE under the shared event names, with the request identity echoed before the stream and a rejected submission answered as a real status code |

Additional results are appended when the matching checklist item is complete.

## Pull requests and versions

- protos: [Duke-ECE/protos#1](https://github.com/Duke-ECE/protos/pull/1) — merged
  (`f3db971`, merged 2026-09-19).
- session-manager: [Duke-ECE/session-manager#1](https://github.com/Duke-ECE/session-manager/pull/1)
  — phases 1 and 2. Open: merging deploys on push to `main`, and the plan puts
  cutover in phase 6, so the merge waits for the coordinated rollover.
- agent-runtime: [Duke-ECE/agent-runtime#1](https://github.com/Duke-ECE/agent-runtime/pull/1)
  — phase 3 in progress.
- managed-agents-backend: [Duke-ECE/managed-agents-backend#1](https://github.com/Duke-ECE/managed-agents-backend/pull/1)
  — phase 4 template lifecycle (D001), request identity.
- managed-agents-frontend: [Duke-ECE/managed-agents-frontend#1](https://github.com/Duke-ECE/managed-agents-frontend/pull/1)
  — phase 5 archive UI.
- Release tags: protos `v0.8.0` (`f3db971`) — `session/v2` + `runtime/v2`
  contracts; the immutable tag consumers pin.

## Cutover and rollback

Finalize before deployment. Require a checkpoint-aware rollback binary, explicit
development-data handling, and successful multi-replica failure tests.
Disabling compaction must preserve hydration of existing checkpoints.

## Outcome and deviations

Pending implementation.
