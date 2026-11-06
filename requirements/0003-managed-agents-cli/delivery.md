# Delivery Plan

Status: In progress

## Work by repository

- `managed-agents-cli`: Go executable, credential storage, OAuth, API commands, tests, README, CI.
- `managed-agents-design`: this requirement and evidence.

## Pull requests and versions

- CLI repository created at https://github.com/Duke-ECE/managed-agents-cli on 2026-11-06. Initial implementation [d91612a](https://github.com/Duke-ECE/managed-agents-cli/commit/d91612a) and Keychain hardening [3105409](https://github.com/Duke-ECE/managed-agents-cli/commit/3105409) are on `main`; no tagged binary release yet.
- Design: [managed-agents-design PR #1](https://github.com/Duke-ECE/managed-agents-design/pull/1), including this requirement alongside the related third-party SDK design.
- Workspace onboarding and pull whitelist: [teamSkills PR #1](https://github.com/Duke-ECE/teamSkills/pull/1), updated with CLI registration.

## Verification

Go 1.25.5: `GOTOOLCHAIN=local go build ./...`, `go vet ./...`, `go test ./...`, and `go test -race ./...` passed. A fake OAuth server test checked the PKCE challenge, callback, code exchange, and credential write; the full production GitHub browser round-trip still requires a human sign-in.

The initial CLI GitHub Actions [CI run](https://github.com/Duke-ECE/managed-agents-cli/actions/runs/42311662267) completed successfully.

Public `go install ...@latest` succeeded through `proxy.golang.org` and the installed binary printed its help. This machine's configured `goproxy.cn` mirror returned 404 for the new module's checksum lookup, so the README records the official-proxy override. The macOS Keychain write path was changed to feed the session through standard input rather than process arguments; a real Keychain round-trip test passed.

Production read-only smoke test with the dedicated member account: email/password `login`, `whoami` (member, not admin), `agents list` (two platform templates), `sessions list` (empty), and `keys list` passed. A chat with the built-in Default assistant streamed `CLI smoke test OK`; `sessions show` returned the persisted turn, then `sessions delete --yes` purged it and `sessions list` was empty. A disposable `cli-smoke-test` API key was created and revoked; `keys list` showed a nonempty `revoked_at`. No secret was written to the repository or to the delivery record.

## Migration, rollout, and rollback

No server migration or deployment. Distribute the CLI only after local gates and a production owner-authenticated smoke test. A release can be withdrawn without changing backend state; issued API keys must be revoked separately.

## Outcome and deviations

Pending.
