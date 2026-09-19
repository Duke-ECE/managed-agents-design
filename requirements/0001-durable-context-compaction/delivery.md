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
- [ ] Reconcile restart/cancellation behavior without replaying completed tools.
  Cancellation and reconnect-dedup are implemented; reconciling a tool call whose
  outcome is unknown after a crash is not.
- [ ] Cover crash boundaries, duplicate delivery, stale leases, compaction races,
  long tool output, missing provider usage, and cache invalidation in tests.
  Handler-level coverage now exists for a completed turn, a duplicate delivery
  (reconnect), a failed model call, and a tool turn exercising both incremental
  boundaries; unit coverage exists for stale leases, lease expiry, compaction
  failures, and usage reporting. Crash-boundary, long-output, and
  cache-invalidation cases do not.
- [x] Pass TypeScript strict build and all Node tests. `npm test` green (127
  tests) under `tsc --strict`.

Partial evidence: `src/canonical.ts`, `src/tokens.ts`, `src/durable-client.ts`,
`src/durable-execution.ts`, `src/compaction.ts`, `src/runtime-events.ts`, and
`src/durable-runtime.ts` (the `runtime.v2.AgentService` handler, registered
alongside v1 and failing closed without a session-manager), with matching tests.
`npm test` green (127 tests) at `c8580ec`, including the handler-level end-to-end
tests.

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
- [ ] Generate or accept a stable client request ID and deterministic request hash
  for every chat submission and reconnect.
- [ ] Expose structured message history and request execution state over HTTP.
- [ ] Replace template mutation/deletion with create, clone, and archive; reject
  new sessions from archived templates while preserving existing sessions.
  Clone and archive are implemented end to end (migration, domain rules, store
  port, PostgREST adapter, HTTP routes, tests) per decision D001: templates are
  immutable, cloning copies configuration into a new private identity, and
  archiving is owner-only, idempotent, and content-preserving. Two parts remain:
  retiring `PATCH`/`DELETE` (a coordinated API+UI change that lands with the
  frontend) and refusing new sessions from an archived template at admission.
- [ ] Resolve and persist one frozen session configuration at admission, using
  credential references rather than transcript secrets.
- [ ] Map revision/state/lease conflicts to stable HTTP/SSE errors and preserve
  cancellation as an allowed operation while a session is busy.
- [x] Pass gofmt, build, vet, unit tests, and whole-service integration tests.
  `gofmt -l .` clean, `go build ./...`, `go vet ./...`, `go test ./...`, and
  `./scripts/check.sh` green at the template-lifecycle commit.

Exit criterion: browser requests are idempotent, session admission is race-safe,
and templates/configuration cannot silently mutate a running session. Partially
met: templates can no longer mutate a session through the new lifecycle, but
admission is not yet race-safe because session creation still runs on v1.

### Phase 5: frontend structured history

- [ ] Render canonical ordered blocks with tool-call/result associations.
- [ ] Show queued, running, completed, failed, cancelled, interrupted, partial,
  and provisional states without presenting unsaved output as durable.
- [ ] Generate and reuse stable request IDs across SSE reconnects.
- [ ] Replace Edit/Delete template actions with Clone/Archive and filter archived
  templates from the default new-session picker.
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
| 2026-09-19 | managed-agents-backend | template lifecycle | `gofmt -l .` clean; `go build ./...`; `go vet ./...`; `go test ./...`; `./scripts/check.sh` — clone/archive rules, adapter, HTTP routes, and errors-to-status mapping covered |
| 2026-09-19 | agent-runtime | `c8580ec` | `npm test` green (127 tests) — tool-boundary persistence verified through the handler (assistant call before dispatch, result before the next model call, link by original id) |

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
  — phase 4 template lifecycle (D001), first slice.
- Release tags: protos `v0.8.0` (`f3db971`) — `session/v2` + `runtime/v2`
  contracts; the immutable tag consumers pin.

## Cutover and rollback

Finalize before deployment. Require a checkpoint-aware rollback binary, explicit
development-data handling, and successful multi-replica failure tests.
Disabling compaction must preserve hydration of existing checkpoints.

## Outcome and deviations

Pending implementation.
