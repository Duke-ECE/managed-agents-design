# Delivery Plan

Status: Implemented locally; device acceptance pending

## Work by repository

- `managed-agents-mobile`: account login, sessions, chat, agents, API keys, administrator access, and `doc/ACCEPTANCE.md`.
- `managed-agents-backend`: add the native callback URL to `supabase/config.toml`.

## Pull requests and versions

The chat timeline and tool activity layout was merged in [managed-agents-mobile PR #1](https://github.com/Duke-ECE/managed-agents-mobile/pull/1) on 2026-11-10. The mobile client now orders tool calls before the final assistant reply and renders each call as one expandable row. Native release and device acceptance remain open.

## Verification

Mobile `npm run lint`, `npx tsc --noEmit`, `npx expo export --platform ios`, and `npx expo export --platform web` passed locally with Node 22. A local sample-data browser preview opened sessions and retained a sent message and reply. Real account, administrator, and native device flows remain for the acceptance checklist.

## Migration, rollout, and rollback

Apply the redirect configuration to Supabase before testing native GitHub login. The mobile app must then be installed as a development build that registers its custom scheme. Revert the redirect configuration and app release separately if the login flow fails; no schema migration is involved.

## Outcome and deviations

Implementation is ready for device and backend acceptance. No live Supabase configuration change or mobile release is claimed.
