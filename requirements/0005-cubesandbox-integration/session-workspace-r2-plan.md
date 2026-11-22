# Session Workspace Persistence with Cloudflare R2

Status: Proposed implementation plan

Date: 2026-11-21 (America/New_York)

## Scope

Keep the existing session as the ownership boundary. Do not introduce a Project entity or move Cube compute to Tencent Cloud. A session that uses tools owns one persistent Cube Volume, mounted at `/workspace`; the MicroVM remains replaceable. Returning to the same **active** session after VM reclamation must recover its files. A new session gets a different volume. An ended session cannot execute tools.

This extends the [existing lifecycle plan](lifecycle-and-clients-plan.md). It preserves `/workspace` files, not arbitrary VM rootfs files or running processes after a cold restart. It does not share files across sessions.

## Verified baseline and storage choice

- The development Cube cluster and manager Go SDK are pinned to `v0.7.3`. The manager currently keeps `session_id -> sandbox_id` in memory, recovers from Cube metadata, and has no volume creation or mount. The 30-minute Cube timeout currently destroys the sandbox.
- Cube's basic S3 Volumes require CubeAPI, CubeMaster, and Cubelet >= 0.6.0; the pinned release meets the version floor. Verify the deployed S3 plugin, `/dev/fuse`, R2 reachability, and actual mount/remount behavior before implementation. The pinned Go SDK already supports `CreateVolume` and `CreateOptions.VolumeMounts`.
- Create one private Cloudflare R2 **Standard** bucket and a bucket-scoped Object Read & Write S3 token. Configure the Cube S3 plugin with the R2 endpoint and signing region `auto`; test path-style addressing with the deployed `s3fs`. Keep credentials on Cube hosts only, never in the MicroVM, browser, transcript, or repository.
- Use the basic Cube S3 Volume API. It isolates volumes under `volumes/<volume_id>/` in the configured bucket. Dynamic bucket/prefix bindings through `private_data` require Cube and SDK >= 0.8.0 and are outside this feature.

Sources: [Cube Volume guide](https://docs.cubesandbox.com/guide/volume), [Cube S3 plugin](https://github.com/TencentCloud/CubeSandbox/blob/master/examples/volume/s3/README.md), [R2 S3 setup](https://developers.cloudflare.com/r2/get-started/s3/), [R2 tokens](https://developers.cloudflare.com/r2/api/tokens/).

## Ownership and lifecycle

Keep file ownership separate from compute state:

| Record | Owner and contents | Lifetime |
| --- | --- | --- |
| `agent_sessions` | session-manager; user, status, transcript | Existing conversation policy. |
| `session_workspaces` (new) | sandbox-manager; session/user IDs, Cube volume ID, state, generation, creation lease, retention deadline, deletion intent and retry metadata | First tool through confirmed volume deletion; never cascade from `agent_sessions`. |
| `session_sandbox_bindings` (already planned) | sandbox-manager; session/sandbox IDs, state, lease, expiry and close retry metadata | Only while compute exists or cleanup is pending. |

Both manager tables must be service-role-only. Store no commands, file bytes, R2 keys, or LLM keys. Transactional reservation and generation fencing must prevent two concurrent first calls from creating two volumes or writable VMs. An uncertain Cube create result must be reconciled before retry; do not assume volume names are unique without verification.

| Trigger | Volume | MicroVM |
| --- | --- | --- |
| First accepted tool | Create once; persist the volume ID before reporting ready. | Create with the volume mounted read-write at `/workspace`. |
| Later tool in the same active session | Reuse the same volume. | Reconnect if live; otherwise create a replacement with the same volume. |
| Idle timeout or runtime restart | Retain. | Reclaim with bounded timeout; a callback is not the only cleanup mechanism. |
| End session | Retain read-only until the session retention deadline and offer owner-scoped export. | Close and retry until gone; reject all further tools. |
| Purge or retention deletion | Persist deletion intent, then delete after detach; retain the cleanup record until confirmed. | Close before volume deletion. |
| R2/Cube outage | Keep the known volume ID and report unavailable; never create an empty replacement volume. | Retry after recovery without claiming a tool ran. |

`session_workspaces` states: `creating`, `ready`, `deleting`, `deleted`, `error`. VM states remain independent. VM loss is recoverable when the volume is intact; missing or inaccessible volume data is a distinct file-loss/unavailable error. Align the file retention deadline with the deployed session-manager retention setting (currently 30 days for ended sessions), rather than hard-coding a second policy. Until export exists, do not promise that ended-session files are user-recoverable.

## Required changes by repository

1. **`sandbox-manager`:** add the workspace store, Cube volume adapter, single-writer lease, generation fencing, quotas, startup reconciliation, retry janitor, and explicit idempotent workspace deletion. Pass `VolumeMounts` when creating Cube sandboxes. `CloseSessionSandbox` must close compute only. Distinguish VM loss from volume loss in status and errors.
2. **`protos`:** add compatible `sandbox.v3` status/delete contract elements if needed; do not redefine `CloseSessionSandbox` to delete the volume. Tag and pin consumers before enabling the new methods.
3. **`session-manager` and `managed-agents-backend`:** verify owner and active status for tool admission. End closes compute but keeps files. Purge/retention leaves a durable deletion intent before the session row disappears. Expose owner-scoped status and bounded export/download for retained ended files; never expose Cube IDs or R2 credentials.
4. **`agent-runtime`:** keep the session ID as the tool scope. Retry a VM reconnection or replacement with the same volume, but never replay a non-idempotent command after an uncertain execution result. Preserve cancellation and persisted tool-result behavior.
5. **Web/mobile:** show starting, ready, temporarily unavailable, and files unavailable states only where needed. Explain that ending stops execution while files remain until the retention deadline; offer export. No Project UI.

## Cost, performance, and recovery gates

- Start with R2 Standard and monitor stored bytes plus Class A/B request counts. The current monthly free tier is 10 GB-month, 1 million Class A, and 10 million Class B requests; egress is free. Proposed pilot limit: at most two workspace volumes per user. Alert at 500 MiB of files per user; do not advertise this as a hard quota because arbitrary `bash` writes can bypass a manager-side byte counter on an `s3fs` mount. Prove filesystem-level enforcement before offering a hard file-size limit. [R2 pricing](https://developers.cloudflare.com/r2/pricing/).
- The Cube S3 driver uses `s3fs`. Benchmark `git status`, small-file writes, dependency installation, and a representative build against local disk. Keep disposable caches out of the persistent volume where the toolchain permits. If the pilot is too slow or request-heavy, keep the manager contract and switch to local SSD execution plus versioned R2 checkpoints before wider rollout. [Cube storage guidance](https://docs.cubesandbox.com/guide/volume).
- R2 objects alone do not recreate a Cube Volume record after Cube control-plane metadata loss. Back up and test restoration of Cube metadata, or test copying an old prefix into a newly created volume. The pinned basic API does not mount an arbitrary existing prefix. `DeleteVolume` recursively deletes the volume prefix: use delayed deletion and a separate backup/export for accidental deletion. [Cube S3 plugin lifecycle](https://github.com/TencentCloud/CubeSandbox/blob/master/examples/volume/s3/README.md).
- Test R2 outage, credential rotation, Cube API/Proxy outage, manager/host restart, duplicate first tools, quota exhaustion, and failed deletion. Alert on stale creates/deletes, orphan volumes, missing IDs, R2 growth, and failed exports.

## Delivery sequence

| Stage | Deliverable | Exit gate |
| --- | --- | --- |
| P0. Storage spike | R2 bucket/token and S3 plugin on the existing Cube host; disposable volume. | Write/read, kill, remount/read, delete, R2 object inspection, latency and request counts recorded. No user traffic. |
| P1. Durable manager | Schema, volume binding, fencing, VM replacement, janitor, metadata restore runbook. | Concurrent ensure, manager restart, expired VM, R2 outage, and failed delete tests pass. |
| P2. Conversation lifecycle | End/purge/retention hooks, status/export API, runtime retry rules. | Active session keeps files across VM loss; ended session rejects tools but permits export; purge cleans files despite temporary outages. |
| P3. Client pilot | Web/mobile states and export in a bounded internal cohort. | Real browser/App flow, owner isolation, quotas, and cost/performance pass. |
| P4. Wider rollout | Alerts, rollback and old-session compatibility. | No stale VM bindings or orphan volumes after staged rollout and rollback drill. |

Keep the existing two-Cube-session pilot cap during P0-P3. Deploy schema before schema-dependent services. Rollback disables new tool admission, then lets the manager drain compute and reconcile volumes; code rollback must not delete R2 objects. Record implementation, tests, measurements, and rollback evidence in [delivery.md](delivery.md).

## Open implementation checks

1. Confirm the deployed `v0.7.3` plugin, `/dev/fuse`, and R2 network path.
2. Verify R2 signing region and addressing with the bucket-scoped token without exposing secrets in logs.
3. Measure a representative repository before committing to direct `s3fs` mount for general use.
4. Choose an owner-scoped export format and size cap that does not re-enable tool execution for ended sessions.
