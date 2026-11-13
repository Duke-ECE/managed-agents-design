# 0006: Thinking Mode Control

Status: Implementing

Created: 2026-11-13

## Goal

Let PolyAgent users choose a model's thinking mode once when creating an agent
template or session, then reuse that choice for every turn in the session.

## Scope

Add a normalized `thinking_mode` API field, a CLI session-creation flag and
template setting, frozen session persistence, and DeepSeek Flash request
mapping. A message request cannot change thinking mode. The first release
supports explicit modes only for `deepseek-flash` at `api.deepseek.com`.

## Documents

- [Design](design.md)
- [Delivery](delivery.md)
- [Decisions](decisions.md)

## Affected repositories

- `protos`: additive v1 and v2 contract fields.
- `session-manager`: durable session mode and v2 frozen configuration.
- `managed-agents-backend`: API validation, template storage, admission, resume.
- `agent-runtime`: provider request mapping and hydration.
- `managed-agents-cli`: creation-time flag and template JSON support.
- `managed-agents-frontend`: preserve and show a template's selected mode.
