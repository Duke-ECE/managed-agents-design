# Delivery Plan

Status: Not started.

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

## Open implementation details

- Final protobuf and HTTP/SSE schemas.
- Credential storage and key-management migration.
- Coordination of template archival and session-creation admission.
- Provider adapter usage normalization and continuation formats.
- Tool execution reconciliation and fencing capabilities.
- Whether to import existing development histories or perform an explicitly approved reset.

## Verification

No implementation tests have been run. The design's acceptance criteria define
the required checks; record commands and results here during implementation.

## Pull requests and versions

No implementation PRs or release tags yet.

## Cutover and rollback

Finalize before deployment. Require a checkpoint-aware rollback binary, explicit
development-data handling, and successful multi-replica failure tests.
Disabling compaction must preserve hydration of existing checkpoints.

## Outcome and deviations

Pending implementation.
