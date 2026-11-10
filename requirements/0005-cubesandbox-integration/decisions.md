# Decision Log

## D001: Session-scoped Cube workspace

Status: Superseded by D003

Date: 2026-11-08

### Context

The runtime's file/shell tools assume state shared across calls; sandbox.v2 is single-use.

### Decision

The initial proposal used a direct runtime Cube adapter and session-manager-owned mapping.

### Consequences

This would expose Cube integration to the runtime and split sandbox lifecycle ownership.

### Related work

[D003](#d003-sandbox-manager-mediates-the-separate-cube-cluster).

## D002: Dedicated infrastructure feasibility gate

Status: Proposed

Date: 2026-11-08

### Context

The shared VCM cluster is smaller than Cube's published Kubernetes node suggestions; its Kubernetes path is preview.

### Decision

Evaluate Cube on a separate, adequately sized cluster. Keep production tools disabled until measured infrastructure and lifecycle gates pass.

### Consequences

The platform cluster does not host CubeMaster or Cube compute nodes.

### Related work

[Technical design](design.md).

## D003: Sandbox-manager mediates the separate Cube cluster

Status: Proposed

Date: 2026-11-09

### Context

The requested topology places CubeMaster and sandboxes in their own cluster and uses `sandbox-manager` for lifecycle and resource transfer. This changes D001.

### Decision

Keep Cube SDK, credentials, session binding, quotas, resource transfer, and cleanup in `sandbox-manager`. Add session-scoped `sandbox.v3` for `agent-runtime`; preserve v1/v2 during migration.

### Consequences

The manager becomes stateful and needs durable mapping, authentication, reconciliation, and cross-cluster networking. Runtime stays independent of Cube internals.

### Related work

[Technical design](design.md), [delivery plan](delivery.md), [D001](#d001-session-scoped-cube-workspace).

## D004: Defer additional cross-cluster authentication for the development pilot

Status: Accepted for development only

Date: 2026-11-09

### Context

The one-node Cube cluster is reachable from the Duke VCM nodes through the existing NPM HTTPS route and source-IP access list. A private tunnel and a dedicated manager-to-Cube identity would delay the first integration test.

### Decision

Proceed with the reachable HTTPS route for development. Defer a private tunnel, dedicated service identity, and per-service authorization until after the first end-to-end manager integration. Do not widen the current NPM source-IP access list. Keep the existing CubeAPI key enabled and configure the manager to send it. Keep CubeMaster's unauthenticated management port off the public route.

### Consequences

This is a testing exception, not a production security posture. Before broader rollout, replace the public route with a private path or add explicit service-to-service authentication, and verify CubeProxy data-plane access separately. The current IP list authenticates neither a workload nor a user.

### Related work

[Delivery plan](delivery.md), [technical design](design.md).
