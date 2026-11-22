# Provider-Independent Session Workspace Persistence on R2

Status: Proposed implementation plan

Date: 2026-11-21 (America/New_York)

## Goal and boundary

Keep one `/workspace` per session; do not add a Project entity. An active session can resume its files after its sandbox is reclaimed or replaced. The durable source of truth must be independent of CubeSandbox so a later switch to Tencent Cloud AGSX does not require migrating from Cube's Volume records.

Use Cloudflare R2 for **versioned workspace archives**, not as a Cube-owned mounted filesystem. Work happens on sandbox-local disk. The two providers' adapters move an archive in and out using their file-transfer interfaces. AGSX currently documents single-file upload/download and recommends an archive for a directory; do not assume it can mount Cube's Volume. [AGSX file system](https://cloud.tencent.com/document/product/1814/132410), [Cube file/volume capabilities](https://docs.cubesandbox.com/guide/volume).

```text
session-manager ── owns session, workspace revision, authorization, deletion intent
       │ metadata/commit RPC only
       ▼
sandbox-manager ── owns R2 transfer, checkpoint/restore, provider adapters
       ├── Cube sandbox (current)
       ├── AGSX sandbox (future)
       └── R2: immutable workspace archives (canonical files)
```

Session-manager must **not** proxy archive bytes: large transfers would add load and failure coupling to the service that admits requests and stores transcripts. Sandbox-manager reads/writes R2 using a bucket-scoped key from a Kubernetes Secret. No R2 key enters a sandbox, frontend, transcript, or session-manager. The current Cube-host `r2` Volume driver is a P0 feasibility experiment, not the production storage path; remove its host-side credentials and registration after the new path is validated.

## Durable model

Add a `session_workspaces` record owned by session-manager, keyed by `session_id`, containing `user_id`, `state`, `committed_revision`, immutable R2 object key, archive checksum/size, retention deadline, and deletion intent/retry status. Do not cascade it from `agent_sessions`: purge must leave enough information for asynchronous R2 cleanup. Keep service-role-only RLS and expose owner-scoped status through the backend. Never store raw files, commands, or credentials in Postgres.

R2 layout: `sessions/<opaque-session-id>/revisions/<revision-id>.tar.zst`. Upload a new immutable object, verify size and checksum, then atomically advance the committed pointer in session-manager. The pointer is the source of truth; an upload with no committed pointer is garbage-collected after a grace period. Keep prior revisions for a short recovery window and cap total retained bytes. Validate archive paths, symlinks, permissions, file count, and total uncompressed size before extraction. Do not trust an archive solely because it is in the private bucket.

Maintain a separate manager-owned `session_sandbox_bindings` record for the live provider, sandbox ID, lease, expiry, and close retry. A session has at most one writable sandbox and one admitted mutating request at a time. The storage record survives sandbox deletion or manager restart. Cube/AGSX IDs are internal and never used as workspace identifiers.

## Checkpoint and recovery protocol

1. First tool call: validate the user's active session; reserve the sandbox binding; create a provider sandbox with local `/workspace`. No R2 archive exists yet.
2. Resume: read the committed revision from session-manager, download that R2 archive to manager-controlled staging, validate it, transfer it to the new sandbox, and extract it into `/workspace` before accepting tools. A missing or corrupt committed object fails closed; never silently start with empty files.
3. During a request: tools operate on local disk. On completion of the request's tool/assistant sequence, stop new mutations, create a bounded archive, upload it, and atomically commit its revision **before** the request is marked durable or a final `done` event is sent. A request with no file changes can reuse the prior revision after reliable change detection; otherwise checkpoint it.
4. Crash or outage: if a VM dies mid-request, restore the last committed revision in a replacement VM. Mark the in-flight request interrupted; do not replay its shell command automatically. If R2 is unavailable during checkpoint, keep the VM and retry within a bounded deadline. If the VM is subsequently lost, expose that the unfinished request's file changes were not committed.
5. End: reject further tools, checkpoint any completed request, close compute, retain the latest committed archive for owner-scoped export until the session retention deadline. Purge creates a durable deletion intent before deleting the conversation; a janitor removes all session archive objects and retries failures. Align with the deployed session retention policy.

The guarantee is **files from completed requests survive**, not process memory or changes from an interrupted request. External effects of `bash` (for example, a network write) are outside file rollback and are never retried automatically. Tool events and workspace revision commits must share request identity so the transcript cannot claim a file-changing request completed when its checkpoint did not.

## Provider adapters and API work

| Component | Required change |
| --- | --- |
| `session-manager` | Workspace metadata and atomic revision commit with request/generation fencing; owner checks, retention and durable deletion intent. No archive byte stream. |
| `sandbox-manager` | `WorkspaceStore` port with R2 implementation; archive staging, size/checksum validation, upload/restore, cleanup janitor; `SandboxProvider` port for Cube now and AGSX later. Authenticate internal calls. |
| `agent-runtime` | Carry the request/session identity through tool calls; wait for checkpoint success before final completion; distinguish interrupted and storage-unavailable states. |
| `managed-agents-backend` | Owner-scoped workspace status/export; end/purge ordering; never expose R2 credentials or provider IDs. |
| `protos` | Additive workspace checkpoint/status/delete contracts, released before consumers. Keep v1/v2 and current v3 behavior during migration. |
| Web/mobile | Show syncing, ready, temporarily unavailable, and files unavailable states; provide export for ended sessions. No Project UI. |

The future AGSX adapter only needs create/execute/upload/download/close behavior; it does not need Cube Volume semantics. The documented AGSX file API transfers one file at a time, so an archive is the common denominator. [AGSX file-transfer guide](https://cloud.tencent.com/document/product/1814/132410).

## Cost, capacity, and security gates

- Use a private R2 Standard bucket. Its published monthly free tier is 10 GB-month, 1 million Class A and 10 million Class B requests, with free egress. Archive-per-completed-request reduces object operations compared with `s3fs` small-file traffic, but storage and transfer bytes grow with each revision. Measure a representative repository and set maximum archive size, revision retention, and per-user active-session count before rollout. [R2 pricing](https://developers.cloudflare.com/r2/pricing/).
- Sandbox-manager should stream archives with bounded memory and a temporary disk budget; do not buffer the whole archive in gRPC or Postgres. Use a separate Kubernetes Secret for the bucket-scoped R2 key, restrict it to sandbox-manager, and rotate the local Cube-host pilot key after switching paths.
- Back up session-manager workspace metadata and test recovery from a missing/corrupt archive, partial upload, R2 outage, sandbox outage, and manager restart. Verify two concurrent first calls and an end/purge racing a checkpoint. Alert on orphan objects, stale deletion intents, storage growth, and failed exports.
- A full archive after each request may be too slow for large code trees. The pilot must measure upload, download, compression, and restore latency. If it misses the agreed latency budget, add incremental/content-addressed checkpoints behind the same `WorkspaceStore` contract; do not change the ownership model or silently weaken the completed-request guarantee.

## Delivery sequence

| Stage | Deliverable | Exit gate |
| --- | --- | --- |
| P0. Portable archive probe | On the existing Cube host, archive `/workspace`, transfer it to R2 independently of Cube Volume, destroy the sandbox, create another, and restore it. | File checksum and source tree match; no Cube Volume record is required. A READY template is currently missing, so this gate is open. |
| P1. Session metadata | Session-manager schema, atomic revision commits, leases, deletion intent, memory fake and PostgREST tests. | Concurrent commit/fencing, purge race, RLS, and retention tests pass. |
| P2. Manager data plane | R2 store, Cube adapter, archive validation, retry/GC, secret/config and observability. | Crash, R2 outage, corrupt archive, over-limit archive, and orphan cleanup tests pass. |
| P3. Request and client flow | Runtime completion barrier, backend owner APIs, web/mobile states and export. | Real active-session resume, interrupted request, end/export/purge and authorization pass. |
| P4. Provider swap rehearsal | Implement a thin AGSX adapter in a disposable environment. | Restore the same R2 revision into Cube and AGSX and compare files; no production provider switch required. |

Rollout to an internal cohort only after P0-P3 pass. Rollback disables new tool admission but preserves committed R2 objects and workspace metadata; the cleanup janitor remains running. Record actual evidence in [delivery.md](delivery.md). The R2 volume mount P0 experiment already proved host-level attach/detach and read-back, but it does **not** satisfy this provider-independent P0 gate.
