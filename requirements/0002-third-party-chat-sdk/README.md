# 0002: Third-party chat SDK

Status: Draft

Created: 2026-10-29

## Goal

Let a third-party application chat with an existing managed agent through a supported SDK.

## Scope

Version 1 covers selecting an existing agent template, creating a session, sending text, receiving streamed replies, and continuing that session. The SDK does not create or configure agent templates. Sandbox execution is outside this requirement.

TypeScript/JavaScript and user-issued server-side platform API keys are selected. A key acts as its owner and may use that user's accessible existing agents and sessions. Version 1 is text chat; tool execution is a separate platform feature.

## Documents

- [Design](design.md)
- [Delivery](delivery.md)
- [Decisions](decisions.md)

## Affected repositories

- [managed-agents-backend](https://github.com/Duke-ECE/managed-agents-backend): API-key issuance, revocation, and session authentication.
- [managed-agents-frontend](https://github.com/Duke-ECE/managed-agents-frontend): API Keys page.
- [agent-runtime](https://github.com/Duke-ECE/agent-runtime): tool behavior if the selected approach requires runtime changes.
- `managed-agents-sdk/` (local new repository): TypeScript/JavaScript chat client. Package distribution remains open.
- [managed-agents-docs](https://github.com/Duke-ECE/managed-agents-docs): third-party integration guide.
