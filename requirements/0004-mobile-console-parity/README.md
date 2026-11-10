# 0004: Mobile console parity

Status: Implementing

Created: 2026-11-07

## Goal

Give the Expo mobile console the account, session, chat, agent, API key, and administrator workflows described in the mobile repository's `doc/` plans.

## Scope

Mobile UI and API integration, plus the Supabase Auth redirect allow-list for the native GitHub login callback. Backend business contracts and database ownership stay as implemented.

## Documents

- [Design](design.md)
- [Delivery](delivery.md)
- [Decisions](decisions.md)

## Affected repositories

- `managed-agents-mobile`: Expo client and acceptance checklist.
- `managed-agents-backend`: Supabase Auth redirect configuration only.
