# 0001: Immutable Agent Templates

Status: Accepted

Date: 2026-09-18

Implementation: Pending

## Context

Editing a template can make existing sessions change behavior on resume. The
pre-launch platform needs a simple, explicit configuration lifecycle.

## Decision

Templates cannot be edited after creation. Users clone an existing template,
modify the new draft, and save it under a new identity. Old templates may be archived.

Archived templates remain viewable and cloneable, but cannot admit new sessions.
Existing sessions retain their resolved configuration and continue working.
Archival does not revoke credentials or cancel sessions.

Retain ownership and built-in template permissions. Ordinary readers cannot archive
templates they do not own. Hard deletion is not part of the normal workflow.

## Consequences

Template identity provides stable provenance. Session configuration is resolved
and frozen at creation, including platform-default settings. Changing configuration
requires a new template/session rather than mutating a running conversation.
API/UI editing flows must be replaced with Clone and Archive.

## Related work

[Iteration 0001](../iterations/0001-durable-context-compaction/README.md).
