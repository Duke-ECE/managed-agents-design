# 0003: Managed Agents CLI

Status: Implementing

Created: 2026-11-06

## Goal

Provide a Go CLI for browser login and day-to-day management of a user's Managed Agents account, agent templates, sessions, and integration API keys.

## Scope

The first release supports GitHub OAuth in the browser, email/password fallback, `whoami`, agent listing/creation/cloning/archival, session listing/transcripts/rename/end/purge, API key management, and streaming chat. It does not administer other users, change the platform's shared templates, or manage Kubernetes sandboxes.

## Documents

- [Design](design.md)
- [Decisions](decisions.md)
- [Delivery](delivery.md)

## Affected repositories

- [managed-agents-cli](https://github.com/Duke-ECE/managed-agents-cli): Go executable and CI.
- [managed-agents-design](https://github.com/Duke-ECE/managed-agents-design): decisions and delivery evidence.

The CLI consumes existing Supabase Auth and backend HTTP contracts. No backend or frontend change is required for the fixed local callback in the first release.
