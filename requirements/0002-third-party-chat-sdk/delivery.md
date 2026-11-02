# Delivery Plan

Status: In progress

## Work by repository

1. Keep the SDK as a local draft until the backend migration and deployed API are verified; choose a registry before publication.
2. Add any necessary backend contract changes.
3. Implement the SDK and a runnable third-party quickstart.
4. Document errors, cancellation, reconnection, and credential handling.

## Pull requests and versions

- Backend: [PR #2](https://github.com/Duke-ECE/managed-agents-backend/pull/2), branch `feat/api-keys`, commit `41c3cc2` (draft; targets agent-template archive branch).
- Console: [PR #3](https://github.com/Duke-ECE/managed-agents-frontend/pull/3), branch `feat/api-keys-ui`, commits `3357a4e` and `e993bf2` (draft; targets template-archive UI branch).
- Docs: [PR #1](https://github.com/Duke-ECE/managed-agents-docs/pull/1), branch `feat/third-party-chat-sdk`, commit `1328178` (draft).
- Design: [PR #1](https://github.com/Duke-ECE/managed-agents-design/pull/1), branch `feat/third-party-chat-sdk` (draft).
- SDK: local repository, branch `feat/third-party-chat-sdk`, commit `2dba3a4`; no remote, registry, or published version yet.
- Released API-key backend: [PR #3](https://github.com/Duke-ECE/managed-agents-backend/pull/3), merge `8383584`, main deployment [Actions run](https://github.com/Duke-ECE/managed-agents-backend/actions/runs/41388394896) succeeded.
- Released console: [PR #4](https://github.com/Duke-ECE/managed-agents-frontend/pull/4), merge `8e848c8`, main deployment [Actions run](https://github.com/Duke-ECE/managed-agents-frontend/actions/runs/41388757858) succeeded.
- Public developer documentation: [docs PR #3](https://github.com/Duke-ECE/managed-agents-docs/pull/3), merged to main; [Pages deployment](https://github.com/Duke-ECE/managed-agents-docs/actions/runs/41489629746) succeeded. The [API guide](https://duke-ece.github.io/managed-agents-docs/developers/api-guide) and [architecture diagram](https://duke-ece.github.io/managed-agents-docs/diagrams/agent-runtime-concept.png) returned HTTP 200.
- Public landing-page links: [frontend PR #9](https://github.com/Duke-ECE/managed-agents-frontend/pull/9), merged to main; CI passed. Main deployment: [Actions run](https://github.com/Duke-ECE/managed-agents-frontend/actions/runs/41489812093).
- Navigation alignment: [frontend PR #10](https://github.com/Duke-ECE/managed-agents-frontend/pull/10) and [docs PR #9](https://github.com/Duke-ECE/managed-agents-docs/pull/9) were merged to main in that order. The [frontend deployment](https://github.com/Duke-ECE/managed-agents-frontend/actions/runs/41497180242) and [Pages deployment](https://github.com/Duke-ECE/managed-agents-docs/actions/runs/41497553335) succeeded.

## Verification

Navigation alignment: frontend `npm run build` and docs `npm run docs:build` pass. In local preview, `/console` redirected an unauthenticated visitor to `/login` and, after entering preview, landed on `/agents`; `/keys` linked to the public API guide, `/keys?tab=docs` redirected there, and an unknown path showed the 404 page. After release, a fresh production browser visit to `/console` reached `/login`, and `/keys?tab=docs` reached the public API guide. The published docs homepage links to `/console`, and the API guide returns HTTP 200.

The local TypeScript client builds and its four HTTP/SSE/session-management tests pass with Node 22. The repository includes a runnable quickstart for an upgraded backend. Backend `go build ./...`, `go vet ./...`, `go test ./...`, and `./scripts/check.sh` pass, including key lifecycle, PostgREST-adapter, and route-auth tests. Frontend `npm run build` and docs `npm run docs:build` pass. The migration applied cleanly to a disposable local PostgreSQL 16 database; its RLS/grants check returned `t|t|f|f` (RLS enabled, service role permitted, anon/authenticated denied). The disposable database was removed. Production migration `20261102042400_api_keys.sql` was applied to Supabase; the live check returned RLS enabled, service-role access true, anon/authenticated read false, and zero keys. Both release PRs passed CI and their main deployment workflows succeeded. The deployed backend answers health 200, unauthenticated `/api/keys` 401, and invalid platform-key session access 401. A real owner login, key issuance, chat, and revocation smoke test remains pending. D003 fixes text-chat scope; D004 records the staged release. Acceptance cases are in [design.md](design.md).

## Migration, rollout, and rollback

The additive `api_keys` migration, backend, and console were deployed in that order. Run an owner-authenticated smoke test with a disposable key and accessible agent; revoke the key and verify the next request is rejected. Roll back the backend and console deployments if needed; leave the additive table in place until keys are retired. The SDK remains private until its durable cancellation and retry contract is deployed and verified.

## Outcome and deviations

The API-key issuance and console UI subset is live. The complete SDK requirement is still in progress: package distribution, durable cancellation/retry behavior, and an authenticated end-to-end smoke test remain open.
