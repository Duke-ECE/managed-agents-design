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

In progress. Completed items are checked; the remainder is being implemented on
the `docs/link-compaction-design` branch behind
[Duke-ECE/agent-runtime#1](https://github.com/Duke-ECE/agent-runtime/pull/1).

- [x] Pin the released proto tag and sync vendored proto files through the
  repository script.
- [ ] Replace turn-end fire-and-forget persistence with request admission and
  acknowledged incremental durable boundaries.
- [ ] Acquire, renew, and fence a 90-second execution lease; stop writes and
  inference immediately after lease loss.
- [ ] Preserve ordered text/tool blocks, stable tool-call IDs, result links,
  provider continuation metadata, and one usage record per model call.
- [ ] Assemble model context deterministically from frozen config, active
  checkpoint, and retained canonical messages.
- [x] Add a model-aware token estimator and check the input budget before every
  model call, including tool-loop calls. The estimator and budget check exist
  (`src/tokens.ts`) with prefix reuse and baseline invalidation; wiring it into
  the loop is part of the remaining work.
- [ ] Implement safe-cut compaction, summary generation, compare-and-publish,
  mid-request continuation, and summary timeout/failure behavior.
- [ ] Reconcile restart/cancellation behavior without replaying completed tools.
- [ ] Cover crash boundaries, duplicate delivery, stale leases, compaction races,
  long tool output, missing provider usage, and cache invalidation in tests.
- [ ] Pass TypeScript strict build and all Node tests.

Partial evidence: `src/canonical.ts` (versioned platform message model, block and
link validation, pi conversion with links by original tool-call id, incomplete
assistant output excluded from context) and `src/tokens.ts` (input budget,
trigger/target ratios, prefix reuse, conservative fallback) with
`test/canonical.test.ts` and `test/tokens.test.ts`; `npx tsc --noEmit` and
`npm test` green (90 tests) at `5961397`.

Exit criterion: a request can survive process loss at every durable boundary and
long contexts compact without overwriting source messages.

### Phase 4: backend admission and immutable templates

- [ ] Pin the released proto tag and switch session/runtime orchestration to v2.
- [ ] Generate or accept a stable client request ID and deterministic request hash
  for every chat submission and reconnect.
- [ ] Expose structured message history and request execution state over HTTP.
- [ ] Replace template mutation/deletion with create, clone, and archive; reject
  new sessions from archived templates while preserving existing sessions.
- [ ] Resolve and persist one frozen session configuration at admission, using
  credential references rather than transcript secrets.
- [ ] Map revision/state/lease conflicts to stable HTTP/SSE errors and preserve
  cancellation as an allowed operation while a session is busy.
- [ ] Pass gofmt, build, vet, unit tests, and whole-service integration tests.

Exit criterion: browser requests are idempotent, session admission is race-safe,
and templates/configuration cannot silently mutate a running session.

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

Additional results are appended when the matching checklist item is complete.

## Pull requests and versions

- protos: [Duke-ECE/protos#1](https://github.com/Duke-ECE/protos/pull/1) — merged
  (`f3db971`, merged 2026-09-19).
- session-manager: [Duke-ECE/session-manager#1](https://github.com/Duke-ECE/session-manager/pull/1)
  — phases 1 and 2. Open: merging deploys on push to `main`, and the plan puts
  cutover in phase 6, so the merge waits for the coordinated rollover.
- agent-runtime: [Duke-ECE/agent-runtime#1](https://github.com/Duke-ECE/agent-runtime/pull/1)
  — phase 3 in progress.
- Release tags: protos `v0.8.0` (`f3db971`) — `session/v2` + `runtime/v2`
  contracts; the immutable tag consumers pin.

## Cutover and rollback

Finalize before deployment. Require a checkpoint-aware rollback binary, explicit
development-data handling, and successful multi-replica failure tests.
Disabling compaction must preserve hydration of existing checkpoints.

## Outcome and deviations

Pending implementation.
