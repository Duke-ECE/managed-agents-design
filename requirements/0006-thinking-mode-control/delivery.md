# Delivery Plan

Status: In progress

## Work by repository

1. `protos`: add contract fields and tag a new additive release.
2. `session-manager`: persist the effective v1 mode at admission and extend
   the v2 frozen config/hash. Add a timestamped migration.
3. `agent-runtime`: replace the hard-coded DeepSeek Flash override with
   mode-aware request mapping; freeze and hydrate the selected mode.
4. `managed-agents-backend`: add agent-template storage/API and session
   resolution/validation before durable creation; expose the effective mode.
5. `managed-agents-cli`: add the creation-only flag and documentation.
6. `managed-agents-frontend`: display and preserve the template setting.

## Pull requests and versions

Record links and pinned tags as each repository is delivered.

## Verification

Record build/test results and same-prompt live latency/quality evidence here.

## Migration, rollout, and rollback

Apply additive schema changes first. Deploy session-manager and runtime with
legacy-compatible defaults, then backend, then CLI/frontend. Do not enable
new client controls until the server path persists the mode. Roll back by
disabling client controls first; the additive columns and proto fields remain
safe for older binaries. Preserve DeepSeek Flash's existing `off` default
throughout rollout.

## Outcome and deviations

Pending implementation and verification.
