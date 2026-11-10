# Technical Design

Status: Implemented locally; device acceptance pending

## Problem and goals

The mobile starter could sign in with email and show basic sessions and chat, but lacked the web console's management workflows and GitHub login. The six requirements in `managed-agents-mobile/doc/` define the desired mobile behavior.

## Decisions and alternatives

Use Supabase Auth for user identity on both clients. The mobile app sends the user JWT to the existing backend for all platform data and mutations. It does not access business tables or hold service-role credentials. Use Expo Router for the app entry and OAuth callback. GitHub login uses a registered native scheme and therefore requires a development build.

## Data model and contracts

The mobile client uses the existing `/api/me`, `/api/sessions`, `/api/agents`, `/api/keys`, and `/api/admin/users` routes. Session and transcript pages use the backend's `limit`, `offset`, `before_seq`, and `has_more` fields. Backend ownership and administrator checks are authoritative. Agent provider keys are write-only; an API key's full secret is displayed only from the create response.

## Execution and recovery

OAuth returns to `managedagents://auth/callback`, which must be in Supabase Auth's redirect allow-list. The client persists the Supabase session, clears visible private state on sign-out, and keeps completed local chat text visible while waiting for the durable transcript. Interrupted or unconfirmed replies are labeled accordingly.

## Verification

Run mobile lint, TypeScript, iOS export, and web export. Follow `managed-agents-mobile/doc/ACCEPTANCE.md` on a development build with real accounts and the deployed redirect configuration.

## Session discovery and new conversation

Both clients display the full session ID on the session list and match either the title or ID in search. While searching, they request older paginated session-list results so an ID outside the first page can be found. The web console has a signed-in `/console` homepage with a new-conversation entry, recent sessions, and summary counts. On mobile, New conversation first opens a searchable agent picker; selecting an available agent opens the empty chat. The selected agent ID is used when the first message creates the session.

## Open questions

The live Supabase redirect allow-list has not been applied or verified in this local implementation. Native OAuth and cross-client data checks require device acceptance.
