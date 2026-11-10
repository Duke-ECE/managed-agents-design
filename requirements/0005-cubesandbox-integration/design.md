# Technical Design: CubeSandbox Integration

Status: Proposed

## Target topology

```text
Browser -> backend -> agent-runtime
                        | sandbox.v3 gRPC (internal, authenticated)
                        v
                 sandbox-manager (platform cluster)
                    |                 |
                    | PostgREST       | Cube Go SDK
                    v                 v
              session binding     CubeAPI + CubeProxy (separate Cube cluster)
                                      |
                               CubeMaster -> Cubelet -> MicroVMs
```

CubeMaster, CubeAPI, CubeProxy, and Cube compute nodes form a **separate cluster**. The existing `sandbox-manager` stays in the platform Kubernetes cluster and is the only platform service with Cube credentials and network access. It calls CubeAPI for lifecycle and CubeProxy for sandbox command/file traffic, not CubeMaster's internal API. `agent-runtime` sends session-scoped requests to the manager and never sees Cube IDs. Select and pin either Cube's multi-node or dedicated Kubernetes deployment after a proof of concept. [Multi-node architecture](https://cubesandbox.com/guide/multi-node-deploy), [Kubernetes architecture](https://cubesandbox.com/guide/kubernetes/architecture), [Go SDK](https://github.com/TencentCloud/CubeSandbox/blob/master/sdk/go/README.md)

## Research and feasibility

The official Go SDK supports create, connect, commands, file operations, timeout, and kill. Remote clients need both CubeAPI and a CubeProxy route; API reachability alone does not prove command/file access. Verify context cancellation with a pinned SDK/server version. [Go SDK](https://github.com/TencentCloud/CubeSandbox/blob/master/sdk/go/README.md)

Cube's Kubernetes deployment remains preview. Its guide suggests at least 4 CPU/8 GB for a control node and about 16 CPU/32 GB for a compute node, plus dedicated compute capacity, storage, and KVM/PVM host preparation. The team's existing VCM nodes are documented as about 2 CPU/3.5 GB each. The separate cluster avoids competition with platform workloads but still needs its own sizing, virtualization, storage, and upgrade checks. No change to the shared VCM nodes is part of this plan. [Kubernetes overview](https://cubesandbox.com/guide/kubernetes/), [install guide](https://cubesandbox.com/guide/kubernetes/install), [team cluster skill](../../../teamSkills/vm-connection/SKILL.md)

## Existing contracts and new interface

`agent-runtime` has `read`, `write`, `bash`, and `edit` behind `ToolExecutor`, but production uses `NullExecutor`. `sandbox.v2.Execute` accepts a string tool/input and destroys a fresh Pod per call; v1 creates/deletes Pods for ops. Neither represents a durable session workspace, typed files, or Cube resource allocation. Keep v1/v2 intact during migration. Add `sandbox.v3` because changing v2 single-use semantics would be breaking. Tag generated proto code and pin consumers before implementation.

Proposed internal `sandbox.v3.SessionSandboxService`:

| RPC | Semantics |
| --- | --- |
| `EnsureSessionSandbox(session_id, profile_id)` | Idempotently create/reconnect a session MicroVM; return platform state, never Cube ID. |
| `ExecuteTool(session_id, oneof read/write/edit/bash, deadline)` | Typed tool request/result, exit/error and truncation. |
| `PutResource` / `GetResource` | Transfer bounded files into/from `/workspace`; specify chunk and gRPC message limits in the proto. |
| `CloseSessionSandbox(session_id)` | Idempotent termination and retry scheduling. |
| `GetSessionSandboxStatus(session_id)` | Internal absent/creating/ready/lost/closing diagnostics. |

The four built-in tools use `ExecuteTool`; resource transfer supports non-text inputs and generated artifacts. CPU, memory, lifetime, image, and network policy come from a manager-owned, versioned Cube template/profile; callers cannot submit arbitrary values. No direct browser endpoint is proposed.

## Ownership and security

The manager owns a durable `session_id -> Cube sandbox_id` binding with state, creation lease, profile/version, expiry, and cleanup retry metadata. Store it in a manager-owned service-role-only Supabase table via PostgREST, with a memory fake for tests. Atomic reservation/CAS prevents two concurrent first calls creating two live sandboxes. If persistence fails after creation, kill the new sandbox; failed kills enter orphan reconciliation. A janitor retries closing/expired bindings and compares them with Cube's list API. Cube timeout is a final safety bound.

The runtime authenticates to v3 with a dedicated service credential. The manager must verify the claimed session identity or use a trusted session-manager lookup; this proof is a contract-stage decision. Enforce per-user/session quotas in the manager, not just runtime tool whitelist checks. Network-restrict and review existing unauthenticated v1/v2 ops interfaces separately. Cube credentials stay in a manager Kubernetes Secret. Use private, authenticated/TLS routes to CubeAPI and CubeProxy; keep CubeMaster internals private. Never put platform/user LLM keys in a MicroVM. Guest egress starts denied with reviewed exceptions; the platform's `pi-sandbox` NetworkPolicy does not cover a separate Cube cluster. [Network hardening](https://docs.cubesandbox.com/guide/network-hardening), [Go SDK network policy](https://github.com/TencentCloud/CubeSandbox/blob/master/sdk/go/README.md)

## Session lifecycle and tool semantics

1. The first tool call reserves a binding, selects a versioned non-root template, creates the Cube sandbox, and commits its ID. All later calls share `/workspace`. Decide during proto review whether `ExecuteTool` implicitly ensures the sandbox.
2. After manager restart, reconnect from its durable binding. After runtime restart, rebuild only the gRPC client. Serialize mutations per session or prove the durable execution lease covers them.
3. `read` honors line offset/limit; `write` makes parents; `edit` requires unique, non-overlapping matches; `bash` uses non-root user, fixed cwd, hard deadline, and bounded stdout/stderr. Validate/canonicalize paths and reject traversal or symlink escape. A nonzero exit is a tool failure with captured output. Both manager and runtime cap model-facing results.
4. End, purge, expiry, and failed creation trigger idempotent close. Manager cleanup cannot depend only on runtime callbacks. Define maximum workspace lifetime and behavior when Cube loses a workspace; do not silently recreate empty files while the transcript implies they remain.
5. Cancellation propagates through gRPC and Go SDK contexts. Verify that the Cube command actually stops; otherwise terminate the sandbox or wait for its server-side hard timeout before releasing the session lease.

## Verification and acceptance

- Manager tests: idempotent/concurrent create, reconnect, four tools, resource bounds, path/symlink escape, quotas, auth, timeout/cancellation, truncation, close/retry/orphan reconciliation.
- Runtime tests: gRPC tool mapping, errors, whitelist, abort, both runtime modes, no Cube secrets/IDs in SSE or transcript.
- Disposable cross-cluster test: create -> upload/write -> bash/edit/read -> download -> close; two-session isolation; manager/runtime restart; denied egress; CubeAPI and CubeProxy outage separately.
- Operations gate: measured Cube capacity, KVM/PVM feasibility, network/DNS/TLS routes, storage, p95 latency, and cleanup success. Pilot with an internal cohort.

## Open questions

1. Which separate Cube topology, node sizes, and private routes are available?
2. What is the session identity proof for manager RPCs, and who gets quota?
3. What are resource byte limits and workspace persistence expectations?
4. How do create metadata, list/filter, and cancellation work in the pinned Cube release?
