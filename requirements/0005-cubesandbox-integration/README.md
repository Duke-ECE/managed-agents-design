# 0005: CubeSandbox Integration

Status: Implementing

Created: 2026-11-08

## Goal

Enable an agent session to run `read`, `write`, `bash`, and `edit` in an isolated CubeSandbox MicroVM, with a consistent workspace across tool calls and explicit cleanup.

## Scope

Research and a staged integration of a separate Cube cluster mediated by the platform's `sandbox-manager`, including resource transfer, durable session mapping, security controls, verification, rollout, and rollback. A single-node Cube development cluster is running on `UbuntuServer`; production tool execution has not been enabled.

## Documents

- [Two-cluster architecture diagram](two-clusters.png)
- [Technical design](design.md)
- [Delivery plan](delivery.md)
- [Session lifecycle and web/mobile development plan](lifecycle-and-clients-plan.md)
- [Decisions](decisions.md)

## Affected repositories

- `agent-runtime`: gRPC tool bridge and session lifecycle calls; no Cube SDK or credentials.
- `sandbox-manager`: Cube Go SDK adapter, durable session mapping, resource transfer, quotas, and cleanup.
- `protos`: new session-scoped `sandbox.v3` contract.
- `managed-agents-design`: this requirement and delivery evidence.
- `session-manager`: possible session identity/status lookup for manager authorization and cleanup; exact interface remains open.

## Research sources

- [CubeSandbox Kubernetes overview](https://cubesandbox.com/guide/kubernetes/)
- [Helm installation and prerequisites](https://cubesandbox.com/guide/kubernetes/install)
- [Official Go SDK](https://github.com/TencentCloud/CubeSandbox/blob/master/sdk/go/README.md)
- [Multi-node deployment](https://cubesandbox.com/guide/multi-node-deploy)
- [Kubernetes architecture](https://cubesandbox.com/guide/kubernetes/architecture)
