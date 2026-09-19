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
- [x] Implement safe-cut compaction, summary generation, compare-and-publish,
  mid-request continuation, and summary timeout/failure behavior. Safe-cut
  selection only summarizes a prefix whose tool calls are all resolved, the
  summary is deadline-bounded with one retry, publication is a compare-and-publish
  through the caller's lease that leaves the old checkpoint active on any failure,
  and `prepareNextTurnWithContext` hands the compacted context back to the same
  request. Unit-tested, wired, and exercised through the handler; a handler-level
  test of compaction *specifically* does not exist.
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
structured history, and archived-template admission are done.

- [x] Pin the released proto tag and switch session/runtime orchestration to v2.
  The tag is pinned at `v0.8.0`; `session.v2` serves canonical history reads and
  canonical admission, and the chat path runs on `runtime.v2` — all behind
  `CANONICAL_SESSIONS`, the single cutover switch that keeps the v1 path
  authoritative until the cutover is rehearsed. Durable turns stream over SSE
  under the same event names as v1.
- [x] Generate or accept a stable client request ID and deterministic request hash
  for every chat submission and reconnect. `Service.ChatTurn` derives the identity
  and passes it to `runtime.v2`, whose request root carries `client_request_id`
  and `request_hash`, so session-manager deduplicates a reconnect and refuses the
  same id carrying different content. Content is encoded with the canonical binary
  encoding (protojson is not stable across releases) with each block
  length-prefixed. The console generates the id, sends it as
  `X-Client-Request-Id`, records the confirmed id, and offers a resend under it.
- [x] Expose structured message history and request execution state over HTTP.
  `GET /api/sessions/:id/structured` serves the canonical session.v2 history:
  ordered content blocks with tool-call/result links, per-message completeness,
  and the request execution state on the initiating root, paginated like the v1
  transcript. The session slice gained a `CanonicalStore` port, attached through
  a functional option so the existing call sites were untouched, and a missing
  v2 client answers not-implemented rather than an empty history.
- [x] Replace template mutation/deletion with create, clone, and archive; reject
  new sessions from archived templates while preserving existing sessions.
  Creation, clone, and archive are the whole write surface: PATCH is refused with
  a uniform 400 that is produced before any ownership lookup (so it cannot probe
  whether an id exists) and the delete route, handler, service rule, store port
  method, and PostgREST adapter method are gone rather than left unreachable.
  The UI offers Clone and Archive for an own template and View and Clone for a
  platform or archived one, and archived templates are excluded from the
  new-chat picker. Admission revalidates archival (410 before the runtime or
  session-manager is touched) while resume deliberately does not, so a session
  that already resolved a template keeps running.
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
- [x] Map revision/state/lease conflicts to stable HTTP/SSE errors and preserve
  cancellation as an allowed operation while a session is busy. ABORTED (a lost,
  expired, or contended lease; a revision or mutation conflict) maps to 409 and
  INVALID_ARGUMENT to 400, where both previously surfaced as 500; UNAUTHENTICATED
  deliberately stays 500 so an upstream service-token misconfiguration cannot tell
  a browser to re-authenticate. A failed chat stream reports whether it is worth
  retrying, derived from the gRPC code, instead of the hardcoded false that made
  every transient failure look permanent. Cancellation is independent of
  admission: `POST /api/sessions/:id/cancel` drives the runtime's cancel RPC and
  reaches a turn this browser is not streaming — one running on another replica,
  or one left running after a dropped connection — which aborting a local fetch
  cannot. The owning runtime still writes the terminal state with its partial
  output, so a cancelled turn is never presented as completed.
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
- [x] Preserve cursor pagination, cache revalidation, cancellation, and history
  navigation under the v2 response schema. All four are contract-agnostic at the
  UI level and were left untouched: the same `limit`/`before_seq` window, the same
  SWR cache and background revalidation, the same abort-on-Stop, and the same
  Load-earlier navigation. The one thing the schema change did break is fixed: a
  cancelled or interrupted turn is persisted as an assistant message with status
  partial or interrupted, which the flat transcript route cannot express, so
  reloading presented unfinished output as complete. The console reconciles the
  canonical record on `seq` — both routes number canonical messages identically —
  and marks those turns in the background after the transcript renders. A
  deployment without session.v2 simply returns nothing to mark.
- [x] Pass the TypeScript and Vite production build. `npm run build`
  (`tsc -b && vite build`) green at every frontend commit in this requirement.
- [ ] Run a multi-service local integration covering create, chat, tool loop,
  resume, cancel, end, delete, and template archive/clone. Partially run: two
  services were driven together over gRPC and create, chat, the tool loop,
  duplicate delivery, and restart hydration are verified (see the phase 6
  integration run below). Cancellation, end/delete, and template archive/clone
  were not exercised, so the item stays open.
- [ ] Run multi-replica failure tests for runtime death, lease expiry/takeover,
  duplicate request delivery, and stale mutation rejection. Run and verified:
  concurrent rejection while a session executes, replica death leaving the root
  durably running, lease expiry, termination of the orphan as interrupted, and
  recovery on another replica; duplicate delivery was verified earlier. Stale
  mutation rejection is unit-tested. A replica *taking over* the orphaned lease to
  finish it was not exercised, so the item stays open.
- [ ] Run long-session tests that trigger between-request and mid-request
  compaction, then verify deterministic hydration after restart. Mid-request
  compaction is verified across services (see the long-session compaction run
  below): a turn over a small-window session compacted and published a checkpoint
  while retaining every original message. Between-request compaction and
  hydration *from a checkpoint* after a restart were not separately exercised, and
  the run's two bug fixes show how much the live path differed from the tested
  one.
- [x] Verify owner isolation, service-token paths, RLS denial, credential
  non-disclosure, and log/fixture secret scanning. Verified: the database suite
  asserts on real Postgres that RLS is enabled with no anon/authenticated
  policies on every service-owned table and that the transaction functions are
  executable by the service role alone; session-manager unit and integration
  tests cover owner-scoped reads and writes, cross-user denial, and the
  token-only paths; the backend asserts the resolved API key appears nowhere in
  the frozen configuration; and both services' `check.sh` now scans *every*
  committed file — not just manifests, which could not see a key pasted into
  source or a fixture — for OpenAI-, Supabase-, Google-, and GitHub-shaped
  literals, passing on the tree as it stands. What is not covered: a scan of
  emitted log lines for message bodies, which is enforced structurally in the
  telemetry record shape rather than checked.
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
| 2026-09-19 | managed-agents-frontend | `10f0770` | `npm run build` green — interrupted turns stay marked across a reload by reconciling the canonical record on seq |
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
| 2026-09-19 | managed-agents-backend | `aea65bc` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...` (10 packages); `./scripts/check.sh` — hard deletion retired across route, handler, rule, port, and adapter |
| 2026-09-19 | managed-agents-backend | `6a67872` | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...` (10 packages); `./scripts/check.sh` — cancellation independent of admission, refusal reported as a conflict |
| 2026-09-19 | session-manager | `0005144` | Secret scan widened to every committed file; `./scripts/check.sh` green |
| 2026-09-19 | managed-agents-backend | `320b3f2` | Secret scan widened to every committed file; `./scripts/check.sh` green |
| 2026-09-19 | managed-agents-frontend | `11f6032` | `npm run build` green — cancel offered for a turn running elsewhere |
| 2026-09-19 | managed-agents-backend | `b8c6872` | Guide updated: no delete route documented |
| 2026-09-19 | managed-agents-backend | `45671f1` | Guide updated: no delete route, PATCH described as a uniform immutability refusal |
| 2026-09-19 | managed-agents-frontend | `f73c048` | `npm run build` green — the dead `deleteAgent` client removed |
| 2026-09-19 | managed-agents-backend | `d4c5b17` | Durable chat streamed over SSE under the shared event names, with the request identity echoed before the stream and a rejected submission answered as a real status code |

### Phase 6 integration run (2026-09-19)

Two services were run locally together and driven over gRPC: session-manager in
its in-memory mode (`:50053`, service token `tok`) and agent-runtime on
`:50055` (the default `:50052` is held by macOS `rapportd`), pointed at a
throwaway OpenAI-compatible endpoint on `:5099` that streams a canned
completion.

Verified across the wire:

- **Canonical admission.** `session.v2.CreateSession` with a frozen
  configuration returned a session whose config the runtime then used.
- **Durable chat.** `runtime.v2.Chat` streamed five text deltas, then a `done`
  frame with `EXECUTION_STATUS_COMPLETED`, revision 3, and aggregate usage
  11/5/16.
- **Durable persistence.** The transcript read back from session-manager held the
  request root (client request id, request hash, execution status, started and
  finished timestamps) and the assistant message with its per-model-call usage,
  at revision 3.
- **Duplicate delivery.** Replaying the same `client_request_id` returned the
  *same* root id and status and left the transcript at two messages: no second
  model call, no duplicate rows.
- **Restart hydration.** After killing and restarting the runtime, a new turn
  appended to the same transcript (sequences 1-4) and the runtime logged
  `hydrated durable session <id> with 2 canonical messages`. The telemetry sink
  emitted a `durable_write` record for the finish (mutation id, message count,
  latency, ok).

**Tool loop.** A second run with an endpoint that emits an OpenAI tool-call chunk
drove a full tool cycle: the runtime streamed a `tool_call` frame (`call_1`,
`bash`) and a `tool_result` frame for the same id, made a second model call, and
completed. The durable transcript holds the whole turn — the root, the assistant
message carrying the `tool_call` block, the tool message carrying its
`tool_result` with `reply_to_message_id` pointing at that assistant message, and
the final assistant text. Each model call kept **its own** usage record (20 then
30 input tokens), which is the design's per-call accounting rather than an
aggregate.

The first attempt at this produced no tool frames; the fault was in the throwaway
endpoint, which tested message content for a string when the client sends an array
of parts. Fixing the harness produced the run above.

**Multi-replica failure run.** Two agent-runtime replicas (`:50055`, `:50056`)
were pointed at one session-manager, with an endpoint that stalls when the prompt
asks it to so a turn stays in flight.

- *Concurrent rejection.* With replica A holding a live lease mid-turn, replica B
  was refused with `FAILED_PRECONDITION: another request is already active on this
  session`, and the durable root showed `EXECUTION_STATUS_RUNNING`. A different
  request cannot be admitted while the session executes.
- *Replica death.* Killing A mid-turn left the root durably `RUNNING` — the record
  never fabricates a completion for a turn nobody finished.
- *Lease expiry.* After the 90-second lease lapsed, the owner's `CancelRequest`
  terminated the orphan as `EXECUTION_STATUS_INTERRUPTED` with
  `cancellation_requested`, which is the design's rule for an unknown outcome:
  surface it and require an explicit retry rather than guess.
- *Recovery.* Replica B then admitted a new turn on the same session (the
  transcript advanced to a second root), and B's hydrated context carried the
  earlier turn — the throwaway endpoint matched text that existed only in the
  durable history.

**Long-session compaction run.** A session admitted with a deliberately small
context window (3000 tokens, 200 output limit, so the input budget is 800 and the
trigger 640) was driven through a tool turn with a long input. Compaction fired,
summarized, and published a checkpoint: `covered_through_seq` 1, summary present,
`prompt_version` v1, summarizer model recorded, estimated tokens **1284 before and
49 after**. The transcript still held all four original messages — compaction moved
the context pointer without overwriting history, which is the design's central
promise.

This run found two bugs that no unit test had caught, both since fixed:

1. the tracked canonical view never included the request root, so the input
   estimate under-counted by the whole user message and the budget check fired
   late or never — exactly the long-input case compaction exists for;
2. `publishCheckpoint` built its guard without a mutation hash, so session-manager
   rejected every publication and compaction could never have published at all.

They survived unit testing because those tests either inject the checkpoint
publisher directly (so no real guard is validated) or build the context shape by
hand (so the omission is invisible). A regression test now pins the guard hash.

**Checkpoint-aware hydration after a restart.** The same small-window session was
allowed to publish a checkpoint (`covered_through_seq` 1), the runtime was then
killed and started again as a fresh process, and a new turn was run. The new
process logged `hydrated durable session … with 3 canonical messages` — the
retained post-cutoff set, not the whole history — and its model call carried the
checkpoint summary, confirmed by reading the prompt the provider actually
received: the `[Earlier conversation summary]` block was present and contained the
text the summarizer had produced in the *previous* process. The turn completed at
revision 9.

That is the property the rollback story depends on: a restarted binary hydrates
from durable state plus the active checkpoint, so disabling or losing compaction
does not lose context.

**Not verified by these runs**, and therefore not claimed: a replica *taking over*
the orphaned turn's lease to finish it (the run terminated it through the owner's
cancel instead), end and delete, and template archive/clone — the last of which
needs the backend in the loop. The services were stopped after the runs; this is
evidence, not a running environment.

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
  contracts; the immutable tag consumers pin. No service has been tagged or
  deployed for this requirement.
- Unmerged by design, because merging `main` deploys and the plan puts cutover in
  phase 6: session-manager, agent-runtime, and managed-agents-backend each change
  a contract or a deploy path. All four PRs have green CI.

## Cutover and rollback

Planned, not rehearsed. The requirement's code is complete and green on its
branches, but nothing is deployed and no failure test has been run against a
multi-service environment, so this section is the plan plus its open items rather
than evidence.

### Cutover order

1. Apply the session-manager migrations to Supabase (`supabase db push`). They
   are additive: every existing `agent_sessions` / `agent_messages` row keeps its
   identity, legacy transcript rows survive as `format_version = 0` with
   `last_seq` backfilled, and no row is rewritten or deleted. Nothing reads the
   new tables until step 2, so applying them is inert.
2. Flip `CANONICAL_SESSIONS=1` on the backend. That single switch turns on
   canonical admission (the frozen configuration), the durable chat path on
   `runtime.v2`, and canonical request identity, together. Canonical *reads* are
   already on.
3. Deploy the frontend. It consumes both contracts, so it is safe before or after
   the flip; it must be deployed with or before step 2 only if the v2 `done`
   frame's `aggregate_usage` shape is to render, which the boundary normalizer
   already handles.

The proto tag `v0.8.0` is the contract every consumer pins, and the session-manager
migration is the only schema change.

### Development data

**Decision: retain, do not reset.** The migration was written to preserve existing
development histories, and it does — legacy rows stay readable through the v1
routes and are returned by the canonical routes as `format_version = 0` messages.
A destructive reset is not required by this design and would need its own approved
execution plan. The one thing history cannot recover is the original tool-call
ordering of legacy turns, which was never recorded; those turns keep their stored
order.

### Rollback

Unsetting `CANONICAL_SESSIONS` reverts admission and chat to the v1 path in one
step, with no data loss: the canonical tables are additional state, not a
replacement for the v1 transcript, so sessions created canonically remain
readable through the v1 routes (their transcript turns are durable messages).
Disabling compaction must preserve checkpoint-aware hydration — the runtime reads
the active checkpoint on hydration and only compacts when the budget check fires,
so an operator disabling it leaves existing checkpoints hydrating as before.

A checkpoint-aware rollback **binary** is still required and has not been built or
rehearsed. Until it is, a rollback at the binary level would lose checkpoint
awareness even though it would not lose data.

## Outcome and deviations

The requirement is implemented across five repositories, with green gates in each
and green CI on every open PR, and it is **not delivered**: nothing is deployed
and the cutover has not been rehearsed. The evidence table above records what each
commit verified; the section below records what was deliberately not done.

Final verification sweep, at the commits named in the evidence table:
`gofmt`/`go build`/`go vet`/`go test`/`check.sh` green in session-manager and the
backend (10 packages); `tsc --noEmit` and 149 Node tests green in agent-runtime;
`tsc -b && vite build` green in the frontend; `buf lint`, a breaking-change check
against `main`, and `go build` green in protos; and the durable schema, RLS,
indexes, lease fencing, revision races, and function privileges verified on real
Postgres. Four services were also run together for the integration, multi-replica,
compaction, and hydration runs described above — which is where the two bugs that
no unit test had caught were found.

Deviations from the design as written:

- **`credential_ref` is empty.** The frozen configuration records a reference
  rather than a secret, which is the design's shape, but no private credential
  storage exists yet, so there is nothing to reference; the runtime supplies the
  key from its own environment. The design lists credential storage as an open
  implementation detail.
- **Model limits are constants** (`128000`/`16384`) mirroring what the runtime's
  adapter uses today, not per-model limits. The design explicitly warns against a
  universal 128k constant; recording the values actually in force is the honest
  interim, and per-model limits belong with the templates.
- **Executor-level tool reconciliation does not exist.** The design allows
  reconciling an unknown tool outcome through an executor's status or
  idempotency; no tool executor is wired yet, so the runtime detects and reports
  an unknown outcome and requires an explicit retry instead.
- **Compaction races are guarded, not concurrency-tested.** Parent, config-hash,
  revision, and lease guards are unit-tested; two publishers racing is not.
- **The v1 transcript route remains.** The frontend renders canonical state where
  it matters (streaming, request state, incomplete turns) but still loads history
  through the v1 route. The design asks for a coordinated cutover rather than a
  permanent compatibility projection, so retiring it belongs with the cutover.

What is left to call this complete, in rough order of risk:

1. **Rehearse the cutover and build a checkpoint-aware rollback binary.** Both are
   unstarted, and the rollback path is the one place where being wrong loses data
   rather than time.
2. **Exercise the un-run integration paths**: cancellation through the backend,
   end/delete, template archive/clone with the backend in the loop, between-request
   compaction, and a replica taking over an orphaned lease.
3. **Close the two frozen-configuration gaps** (a credential store and per-model
   limits), which the design lists as open implementation details rather than work
   this requirement can finish alone.
4. **Merge the four PRs in the cutover order above**, which is what deploys them.

The largest lesson from the phase 6 runs is worth recording: four rounds of green
unit tests missed two bugs that made compaction unable to publish, and one live
run found both. The unit tests were not wrong, but they injected the publisher and
hand-built the context shape, so they could not see either defect. Any future work
here should extend the live runs before it extends the unit suite.
