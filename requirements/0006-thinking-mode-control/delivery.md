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

- `protos` [PR #5](https://github.com/Duke-ECE/protos/pull/5) merged and tagged `v0.9.2`.
- `session-manager` v1 [draft PR #2](https://github.com/Duke-ECE/session-manager/pull/2), v2 [draft PR #3](https://github.com/Duke-ECE/session-manager/pull/3).
- `agent-runtime` [draft PR #6](https://github.com/Duke-ECE/agent-runtime/pull/6).
- `managed-agents-backend` [draft PR #6](https://github.com/Duke-ECE/managed-agents-backend/pull/6).
- `managed-agents-cli` [draft PR #2](https://github.com/Duke-ECE/managed-agents-cli/pull/2).
- `managed-agents-frontend` [draft PR #17](https://github.com/Duke-ECE/managed-agents-frontend/pull/17).

## Verification

Local gates passed: proto generation/lint/breaking checks; runtime Node 22
test suite (87 cases); Go build, vet, and test for session-manager, backend,
and CLI; frontend production build; v2 session-manager migrations on fresh
and upgraded disposable PostgreSQL databases. Live same-prompt latency and
quality evidence remains pending.

## Migration, rollout, and rollback

Release order: apply the v1 session-manager migration, then deploy
session-manager with owner redaction, then agent-runtime. Apply the agents
table migration before deploying the backend. Verify the API directly with
fresh DeepSeek Flash sessions at `off`, `low`, and `high`, including a tool call
and restart. Only then release CLI and frontend controls. The v2
session-manager PR targets its separate durable-context branch and is not
part of this production sequence. Roll back by disabling client controls
first; the additive columns and proto fields remain safe for older binaries.
Preserve DeepSeek Flash's existing `off` default throughout rollout.

D003 recovery is staged in the runtime and both session-manager branches:
private `agent_state` rows preserve exact provider messages, token-path
hydration replays them, and owner reads redact them. An integrated fake-provider
test verifies a tool call, runtime restart, and replay of `reasoning_content`
and tool-call IDs. Long histories are read in bounded pages. Production rollout
still requires coordinated migration/deploy order and live latency/quality
evaluation.

## Outcome and deviations

Contract, client plumbing, and private reasoning recovery are prepared in
draft PRs. Coordinated migrations and live evaluation remain.
