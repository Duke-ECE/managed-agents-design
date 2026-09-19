# 0001: Durable Context Compaction

Status: Implementing

Created: 2026-09-18

## Goal

Support durable long-running sessions, accurate message recovery, and automatic
context compaction across runtime replicas.

## Scope

Canonical messages, request-root execution state, incremental persistence,
execution leases, context checkpoints, immutable templates, and structured history.
Cold storage, branching, and cross-session memory are excluded.

## Documents

- [Technical design](design.md)
- [Delivery plan](delivery.md)
- [Immutable templates decision](decisions.md#d001-immutable-agent-templates)

## Affected repositories

- [protos](https://github.com/Duke-ECE/protos)
- [session-manager](https://github.com/Duke-ECE/session-manager)
- [agent-runtime](https://github.com/Duke-ECE/agent-runtime)
- [managed-agents-backend](https://github.com/Duke-ECE/managed-agents-backend)
- [managed-agents-frontend](https://github.com/Duke-ECE/managed-agents-frontend)

The approved design is being implemented contract-first across the affected
repositories. Delivery evidence and deviations are tracked in `delivery.md`.
