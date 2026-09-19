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
- [ ] Merge the proto PR, publish a semver tag, and record the immutable tag here.

Exit criterion: consumers can pin a reviewed v2 contract tag. Until then,
consumer branches may test against a temporary local module replacement, which
must never be committed.

### Phase 1: Durable storage foundation

- [ ] Add a non-destructive migration for canonical message fields, request-root
  state, session revisions, fenced leases, immutable session configuration,
  checkpoints, and idempotency receipts.
- [ ] Add and validate indexes for latest-window history, request history,
  request deduplication, and one nonterminal request per session.
- [ ] Add short Postgres transaction functions for request admission, lease
  operations, incremental append, terminal completion, cancellation, and
  checkpoint publication.
- [ ] Verify migrations from both an empty database and the existing v1 schema
  with retained rows.
- [ ] Verify RLS remains enabled with no anon/authenticated policies on every
  service-owned table.

Exit criterion: all durable state transitions are atomic inside Postgres and can
be invoked through PostgREST without a read-max-write sequence race.

### Phase 2: session-manager v2

- [ ] Add v2 domain types and validation without importing transport or storage.
- [ ] Add a behavior-parity in-memory store for the expanded persistence port.
- [ ] Implement PostgREST adapters for transactional RPCs and cursor reads.
- [ ] Register `session.v2.SessionService` alongside v1 during migration.
- [ ] Enforce owner-versus-service-token authorization and credential redaction.
- [ ] Cover deduplication, hash conflicts, request state transitions, revision
  conflicts, lease fencing/expiry, immutable messages, safe checkpoints,
  pagination, end/delete races, and cancellation with unit tests.
- [ ] Pass `scripts/check.sh`, gofmt, build, vet, unit tests, and whole-service
  integration tests using Go 1.25 with `GOTOOLCHAIN=local`.

Exit criterion: session-manager is a complete privilege and persistence boundary
for the v2 state machine, while v1 remains usable for controlled migration.

### Phase 3: agent-runtime durable execution

- [ ] Pin the released proto tag and sync vendored proto files through the
  repository script.
- [ ] Replace turn-end fire-and-forget persistence with request admission and
  acknowledged incremental durable boundaries.
- [ ] Acquire, renew, and fence a 90-second execution lease; stop writes and
  inference immediately after lease loss.
- [ ] Preserve ordered text/tool blocks, stable tool-call IDs, result links,
  provider continuation metadata, and one usage record per model call.
- [ ] Assemble model context deterministically from frozen config, active
  checkpoint, and retained canonical messages.
- [ ] Add a model-aware token estimator and check the input budget before every
  model call, including tool-loop calls.
- [ ] Implement safe-cut compaction, summary generation, compare-and-publish,
  mid-request continuation, and summary timeout/failure behavior.
- [ ] Reconcile restart/cancellation behavior without replaying completed tools.
- [ ] Cover crash boundaries, duplicate delivery, stale leases, compaction races,
  long tool output, missing provider usage, and cache invalidation in tests.
- [ ] Pass TypeScript strict build and all Node tests.

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

Additional results are appended when the matching checklist item is complete.

## Pull requests and versions

- protos: [Duke-ECE/protos#1](https://github.com/Duke-ECE/protos/pull/1)
- Release tags: pending reviewed merges.

## Cutover and rollback

Finalize before deployment. Require a checkpoint-aware rollback binary, explicit
development-data handling, and successful multi-replica failure tests.
Disabling compaction must preserve hydration of existing checkpoints.

## Outcome and deviations

Pending implementation.
