# Technical Design

Status: Implementing

## Problem and goals

The console exposes user-scoped API operations, but developers need a terminal workflow akin to GitHub CLI. The CLI must log in interactively without asking users to copy browser tokens, keep refresh tokens out of source trees, and exercise the same owner-scoped APIs as the console.

## Decisions and alternatives

- Go 1.25 and a standalone `Duke-ECE/managed-agents-cli` repository; stdlib-first CLI and HTTP client.
- Browser login uses Supabase GitHub OAuth with PKCE and the already allow-listed `http://localhost:5173` callback. The CLI listens only on loopback and exchanges the returned code itself. The fixed port must be free; a clear error is shown if it is occupied.
- Email/password login is a terminal fallback. The password is read without echo and is never persisted. A refreshed Supabase session is saved in macOS Keychain; non-macOS hosts use a mode-0600 user config file until platform credential-store support is added.
- User JWTs authenticate account, agent, key, and session operations. Integration API keys remain server-side third-party credentials and are not used to manage agents or keys.
- Agent templates are immutable after creation: users clone and archive instead of editing. Platform templates are read-only and may be cloned.

## Data model and contracts

The CLI stores the access token, refresh token, and expiry for one account. It obtains the Supabase URL and public anon key from built-in public defaults, with environment overrides for staging. It never stores service-role keys or LLM provider keys. Backend requests use `Authorization: Bearer <user JWT>` and the existing `/api/me`, `/api/agents`, `/api/sessions`, and `/api/keys` routes. JSON responses remain snake_case. The CLI handles SSE `text_delta`, `error`, and `done` frames.

## Execution and recovery

On each command, the CLI loads the saved session, refreshes it before expiry, persists rotated tokens, then makes the backend call. A 401 yields a login instruction. Browser login has a finite timeout, closes its listener, rejects callback paths other than `/`, and only exchanges a PKCE code. Logout revokes only the local Supabase session and removes local credentials. Destructive archive, purge, and revoke commands require `--yes` or interactive confirmation.

## Verification

Unit tests cover PKCE callback and token exchange against fake HTTP servers, session refresh, credential storage permissions, JSON API errors, and SSE parsing. An owner-authenticated smoke test checks `login -> whoami -> agents list -> sessions list -> logout` against production without creating data. Mutating smoke tests use disposable agents, sessions, and keys and clean them up.

## Open questions

- Whether to support more than one named account/profile in a later release.
- Packaging beyond `go install` and downloadable release binaries.
