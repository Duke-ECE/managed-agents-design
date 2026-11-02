# Delivery Plan

Status: In progress

## Work by repository

1. Keep the SDK as a local draft until the backend migration and deployed API are verified; choose a registry before publication.
2. Add any necessary backend contract changes.
3. Implement the SDK and a runnable third-party quickstart.
4. Document errors, cancellation, reconnection, and credential handling.

## Pull requests and versions

None yet.

## Verification

The local TypeScript client builds and its four HTTP/SSE/session-management tests pass with Node 22. The repository includes a runnable quickstart for an upgraded backend. Backend `go build ./...`, `go vet ./...`, and `go test ./...` pass, including key lifecycle, PostgREST-adapter, and route-auth tests. Frontend `npm run build` and docs `npm run docs:build` pass. The migration applied cleanly to a disposable local PostgreSQL 16 database; its RLS/grants check returned `t|t|f|f` (RLS enabled, service role permitted, anon/authenticated denied). The disposable database was removed. The migration has not been applied to Supabase and no deployed integration test has run. D003 fixes the first release to text chat. Acceptance cases are in [design.md](design.md).

## Migration, rollout, and rollback

Apply the additive `api_keys` migration, deploy the backend, then deploy the console and run the quickstart with a disposable key and accessible agent. Revoke that key and verify the next request is rejected. Roll back the backend and console deployments if needed; leave the additive table in place until keys are retired. Do not publish the SDK before a deployed backend supports its contract.

## Outcome and deviations

Pending.
