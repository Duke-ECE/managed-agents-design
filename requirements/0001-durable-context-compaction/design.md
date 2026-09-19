# Durable Sessions and Context Compaction

Status: Proposed; implementation has not started.

## Objective

Design a coherent pre-launch architecture rather than preserving the current
transcript format. Store one canonical message history and derive both browser
display and model context from it. Compaction preserves session identity and
never overwrites original messages.

All documentation, code comments, prompts, and interface descriptions are English.
Cold storage, branching, and cross-session memory are outside this proposal.
No development data is deleted by adopting this proposal. A destructive reset
requires a separately agreed execution plan.

## Core decisions

- Use one authoritative message table: `agent_messages`.
- Preserve ordered content blocks, original tool-call IDs, and result references.
- Store request identity and execution state on the initiating user message.
  A turn is a logical message group, not a separate table.
- Persist user input and completed model/tool messages incrementally.
- Preserve interrupted and failed turns rather than silently discarding them.
- Use renewable fenced leases for cross-runtime execution ownership.
- Build model context deterministically from committed data; memory is a cache.
- Keep database access through PostgREST as required by platform standards.

There is no separate display transcript or context-message table. UI bubbles are
a projection of canonical messages grouped by request, not the storage format.
There is no `agent_turns` table. Session configuration and compaction
checkpoints remain separate because they are not conversation messages.

## Accepted product decisions

- Failed/cancelled requests retain user input and completed tool results. Context
  explicitly records interruption and excludes incomplete assistant replies.
- Support compaction inside a running request at durable, fully resolved tool
  boundaries, as well as between completed requests.
- Templates are immutable after creation. Changes create a new template by
  cloning; old templates can be archived.
- Reject a different incoming request while the session is executing or compacting.
  Cancellation remains available. Request queuing and steering are out of scope.

## Responsibilities

| Component | Responsibility |
| --- | --- |
| agent-runtime | Execute turns, budget tokens, summarize, assemble provider requests |
| session-manager | Authorization, canonical persistence, leases, revisions, checkpoints |
| backend | User authentication, stable request IDs, HTTP/SSE transport |
| frontend | Render structured messages, tool associations, partial and interrupted states |
| Postgres through PostgREST | Atomic mutations, constraints, idempotency, fencing |

Use versioned platform message types and Pi/provider adapters, not unversioned SDK
objects. Verify hooks against the installed pinned Pi package. Do not depend on
the moving local Pi checkout or import the entire coding-agent CLI.

## Data model

These are logical field specifications, not executable migration SQL.
Use composite foreign keys for same-session references. All session-owned
records cascade on permanent session deletion.

### agent_sessions

Keep identity, ownership, template reference, title, status, and lifecycle dates.
Add:

- `last_seq`: highest allocated canonical sequence, initially zero.
- `revision`: monotonic semantic state version.
- `active_checkpoint_id`: same-session checkpoint reference.
- `active_request_message_id`: initiating user message of the active request,
  if any; this is a reference, not a second copy of its execution state.
- `lease_owner`, `lease_generation`, `lease_expires_at`.

Lease renewal uses database time and does not increment semantic revision.
Messages, request-state transitions, and checkpoint publication do.

### agent_messages

- `session_id`, `seq`: unique canonical order; index on both fields.
- `id`: stable message ID, unique within the session.
- `request_message_id`: same-session reference to the initiating user message.
  The initiating message references itself; every generated message references it.
- `role`: user, assistant, or tool.
- `content`: ordered JSONB content blocks.
- `format_version`: versioned serialization contract.
- `message_status`: complete, partial, or interrupted; this describes message
  content completeness, not the execution status of the entire request.
- `reply_to_message_id`: optional causal reference for tool results.
- `usage`, `provider_metadata`, `created_at`.

Only initiating user messages carry these request-level fields:

- `client_request_id`, `request_hash`: request identity and deduplication.
- `execution_status`: queued, running, completed, failed, cancelled, or interrupted.
- `started_at`, `finished_at`, `error_code`: execution lifecycle metadata.

Other messages leave these fields null. Enforce this shape using CHECK constraints.
Use a same-session self-referencing foreign key for `request_message_id`; mutation
functions additionally validate that the target is a self-referencing user root.
Use a partial unique index on `(session_id, client_request_id)` for request roots,
and an index on `(session_id, request_message_id, seq)` for request history.
Enforce at most one nonterminal request root per session with a partial unique
index and serialized session-row mutations.

The same client request ID and payload return the existing root and execution
state. Different content with the same ID is rejected. An explicit retry after
failure creates a new request root with a new ID; reconnecting does not rerun
tools. The root's sequence is the group's starting boundary; derive the final
boundary from its associated messages instead of duplicating sequence ranges.

Request lifecycle metadata may change through validated state transitions;
completed message content and causal links remain immutable. For example, a
fully persisted user message has `message_status = complete` while its
`execution_status = running`. These fields describe different things.

Example blocks:

```json
{"type":"text","text":"I will inspect the project."}
{"type":"tool_call","id":"call_123","name":"bash","arguments":{"command":"pwd"}}
{"type":"tool_result","tool_call_id":"call_123","status":"success","text":"/workspace"}
```

An assistant message may contain text and multiple tool calls. Link parallel
results by ID, never adjacency or tool name. Allocate sequence numbers in
persistence order; provider adapters reconstruct required protocol ordering
using explicit links. Validate role/block combinations, unique tool-call IDs
within a session, and result references.

Preserve provider continuation fields, including opaque signatures when required.
Keep credentials and transport objects out of message payloads. Do not assume
that assistant text alone is sufficient for replay.

Completed message content is immutable. Persist partial snapshots when cancelling;
raw token deltas remain transport events rather than database rows per token.
A sudden crash may lose provisional deltas since the last durable boundary.
The UI must distinguish provisional output from committed history.

### agent_session_config

- `session_id`: primary key; one immutable resolved configuration per session.
- Effective system prompt, model provider/ID/base URL.
- Actual context window, output limit, tool definitions or versioned references.
- `credential_ref`, `created_at`.

System/config records are not chat messages. Resolve and freeze configuration at
session creation. All requests use this snapshot; configuration changes require a
new template and a new session. Store a format version/hash for decoding and cache
validation, not a mutable configuration-revision history.

Credential values belong in private credential storage, never transcript,
summaries, logs, or browser responses. Implement credential references with an
explicit secret-storage/key-management design; moving plaintext to another
JSON field does not provide that protection.

### Immutable templates and archival

Creation stores a new template identity and immutable name, system prompt, model
selection, tool selection, and credential reference. Clone creates an editable
draft which becomes a new immutable template on submission. An optional
`source_template_id` records provenance without runtime inheritance.

Archive changes lifecycle metadata such as `archived_at`, not template content.
Archived templates remain viewable and cloneable but are excluded from the default
new-session picker. Existing sessions remain usable with their frozen configuration.
Revalidate archival status server-side when admitting session creation. Coordinate
archive and creation admission: an already admitted creation may finish; a later
admission is rejected. This cross-service boundary requires an explicit protocol,
not just a frontend filter.

Replace template editing with Clone and Archive in the API/UI. Omit hard deletion
from the normal workflow. Preserve ownership and built-in template permissions;
readers do not gain archival rights. Archive does not revoke credentials or cancel
sessions. Credential rotation/revocation is a separate operational action and must
not silently switch a session's provider/model. Resolve platform-default model
settings once at session creation and include them in the frozen snapshot.

### agent_context_checkpoints

- `session_id`, `id`: composite identity.
- `parent_checkpoint_id`: previous same-session checkpoint.
- `covered_through_seq`: inclusive original-message cutoff.
- `source_head_seq`, `source_revision`, configuration format/hash.
- `active_request_message_id`: nullable reference for mid-request compaction.
- `resume_after_seq`: durable execution boundary captured during generation;
  retained messages may lie between the summary cutoff and this boundary.
- `summary`, `format_version`, `prompt_version`, summarizer model identity.
- Estimated token counts before/after, generation usage, creation time.

Persist only successful checkpoints. They are immutable; activation changes the
session pointer atomically. Derive the retained range as
`seq > covered_through_seq`. A checkpoint summarizes the complete prefix using
its parent summary plus newly covered original messages.

## Context construction

```text
Model input = effective system instructions and tools
            + active summary
            + eligible original messages after its cutoff
```

History APIs return original messages regardless of compression. Frontend rendering
groups them by turn while retaining tool relationships. A compaction indicator is
metadata, not an invented assistant reply.

Render summaries as explicitly labeled historical context below system priority.
Never promote quoted user/tool content into system instructions. Preserve goals,
constraints, decisions, completed work, unresolved work, facts, and artifact IDs.
Summaries are lossy and cannot guarantee perfect recall.

Interrupted content stays in history. Exclude incomplete assistant output from
model context by default, with an explicit recovery note explaining interruption.
Retain completed user input and observed tool results. Never fabricate a successful
tool result to satisfy a provider protocol.

Cancellation is handled independently of normal request admission, including during
summary generation. Cancel the summarizer and mark the request cancelled through
a fenced transaction. Publication checks execution state and revision so a
cancelled request cannot resume inference after compaction.

## Durable execution

1. Backend validates the user and forwards a stable client request ID.
2. Atomically deduplicate the request, persist its initiating user message with
   queued execution status, and set the session's active request reference.
3. Runtime acquires a renewable fenced lease and hydrates committed state.
4. Persist each completed assistant tool-call message before dispatching tools.
5. Persist each observed tool result before the next model request.
6. Commit final assistant output and the root message's terminal execution status,
   and clear the active request reference in one transaction before SSE done.
7. Release the lease.

Mutation requests carry idempotency keys, hashes, expected revisions, and lease
generations. Allocate sequences, insert messages, and advance revision atomically.
Retry uncertain database outcomes using the same mutation ID, not by rerunning
the model or tool.

### Transaction boundaries

Use short transactions at durable execution boundaries:

| Transaction | Atomic changes |
| --- | --- |
| Accept request | Insert user root, deduplicate request, set active request, advance sequence/revision |
| Begin execution | Acquire lease and move the root from queued to running |
| Save tool calls | Insert complete assistant message before tool dispatch |
| Save tool results | Insert observed results with their call references |
| Finish request | Insert final output, update root execution state, clear active request, release matching lease |
| Publish compaction | Insert checkpoint and update active pointer |

Each message mutation also updates sequence/revision atomically. Cancellation or
failure saves any available partial output and changes the root execution status
in one short transaction. Once terminal, reject new appends to that request.

A turn is a logical execution unit, not a long-running database transaction.
Do not hold BEGIN/COMMIT open across inference or tool execution. Saving the whole
turn only at the end would be atomic but would lose intermediate evidence on a
crash, even when tools already changed external state. A database transaction
cannot roll those external actions back.

Do not infer grouping from database transaction IDs: a request spans several
commits and uses `request_message_id` as its stable business identity. Compression
cutoffs use completed requests or validated message boundaries inside a running
request, always with no unresolved tool calls across the cut.

Track tool execution attempts separately when needed: call ID, attempt ID, state,
executor idempotency key, and timestamps. This is execution metadata, not a second
copy of conversation content. After a crash, a tool call without a confirmed
result has an unknown outcome. Reconcile through executor status/idempotency;
otherwise interrupt the turn and require an explicit retry decision.
Database transactions do not guarantee exactly-once external side effects.

Start with a 90-second lease renewed every 20 seconds. Every runtime mutation
checks generation and expiration. Lease loss stops further execution and rejects
stale writes. End/delete invalidates ownership and prevents late publication.
External executors need separate fencing/cancellation support to prevent stale
process side effects. No database transaction stays open during inference.

## Automatic compaction

Check the budget before every model request, including tool-loop requests:

```text
input_budget = actual_context_window - output_reserve - safety_margin
```

Include system instructions, tool schemas, summary, content blocks, and incoming
input. Use available tokenizers plus a conservative fallback. Calibrate with the
latest individual model request usage, not total turn usage.

### Token accounting and storage

Do not add a `current_tokens` column to `agent_sessions`. Keep
`estimated_input_tokens` in runtime memory, associated with the model, estimator
version, frozen configuration hash, and the exact context used for the last request.
It is a disposable estimate, not authoritative session state.

Persist the usage of each individual model call on its corresponding assistant
message in `agent_messages.usage`. Do not copy the same call usage onto multiple
message rows or combine multiple calls into one assistant record. Aggregate usage
across calls only for cost reporting. Missing usage is unknown, not zero.

Normalize provider usage into total input tokens, including cached input, and
output tokens. Verify each provider adapter's semantics: some SDKs report cached
input separately, while others already include it. Do not omit or double-count
cached tokens when estimating context size.

When the previous request context remains an unchanged prefix, estimate:

```text
next_input_estimate = previous_request_total_input_tokens
                    + estimated newly appended assistant content
                    + estimated new tool results and user messages
                    + estimated incremental message framing
```

Estimate the actual content that will be sent to the provider. Reported output
tokens may include reasoning or other content not replayed verbatim, so they are
not an exact substitute for the serialized assistant-message estimate. The next
response's actual input usage calibrates the estimate again.

Before the first request, after hydration or compaction, and whenever model,
system instructions, tool schemas, or retained context changes, recompute the
entire assembled request instead of reusing an invalid baseline. If a provider
omits usage, use the local estimator with a conservative safety margin.

Check the resulting estimate before every model invocation; do not trigger
compaction solely from the previous response's usage. A newly returned tool
result can increase input substantially. Checkpoints store before/after estimates
for diagnostics, not as a replacement for estimating the next request.

### Trigger policy

Initial tunable defaults:

- Trigger at 80% of input budget; target below 60% after compaction.
- Summary cap of 4,096 output tokens, reduced when the available budget is smaller.
- Prefer retaining the latest two complete turns when they fit.
- A 60-second summary deadline with one transient retry within that deadline.
- Explicit model context/output limits instead of the universal 128k constant.

Prefer completed-request boundaries, but support mid-request compaction for long
tasks. Pause after an assistant message and all associated tool results have been
committed, before the next model call. With parallel tools, wait for every call
in the batch to resolve. Never cut between a tool call and its result or compact
while a tool is executing.

Select a safe cutoff retaining recent complete model/tool exchanges where possible.
The summary must preserve the active user request, constraints, completed actions,
unresolved objectives, and next steps. Keep the root's execution metadata in the
database even if its text is summarized. Do not resend that input as a new request.

Publish under the current lease, recording the active request and durable execution
boundary. Continue the same request with summary plus retained messages; compaction
does not complete it or create a new request. After restart, reconcile tool attempts
and reacquire ownership before continuing. Never replay completed tools. Validate
checkpoint execution metadata against current request state before resuming.

Failed/cancelled requests become eligible only after tool outcomes are reconciled
or explicitly recorded as unknown. Do not hide unresolved work in a summary.

Combine the previous summary with newly covered messages. Process oversized
summary inputs in bounded chunks at safe boundaries; intermediate summaries are
provisional. Budget summarizer requests too. Validate nonempty output, schema,
cutoff, and final token size.

Publish the checkpoint and active pointer atomically, checking parent, source
revision, lease generation, and coverage. Install the context in memory only
after acknowledgement. Failure leaves the old checkpoint active; continue only
if the existing context still fits, otherwise return a compaction error.

For an oversized running turn, cap model-facing tool output with explicit
truncation and original message references while preserving the original result.
Provide authorized range reads of original tool output when needed. If it still
cannot fit even after safe mid-request compaction, return a context-limit error.
An indivisible oversized user input or tool exchange still requires explicit size
handling; compaction does not make the context window unbounded.

On a provider overflow, allow one smaller-context rebuild, then return an error.
Do not rerun previously executed tools as part of this retry.

## Hydration and contracts

Separate history APIs from internal context APIs; both read the same messages.
Internal operations authenticate service callers. User-facing operations enforce
ownership using verified identity, not a caller-supplied bare user ID.

Proposed operations:

- StartRequest: deduplication and durable initiating user message.
- AcquireSessionLease, RenewSessionLease, ReleaseSessionLease.
- GetContextState: settings, checkpoint, revision, message high-water mark.
- GetMessages: bounded sequence-range pagination.
- AppendMessages: fenced, idempotent message-batch commit.
- FinishRequest: atomic final batch and root execution-state transition.
- CommitContextCheckpoint: fenced checkpoint activation.
- GetRequest: root-message lookup by request ID for reconnect and uncertain outcomes.

Hydrate every uncovered message up to the captured high-water mark. Pagination
must not silently drop history. Existing-session load failures return retryable
errors rather than starting empty. Cached runtimes compare durable revision
under their lease before accepting work.

HTTP/SSE expose versioned blocks, request/message IDs, committed sequences, and
root execution status. Provisional deltas reference pending message IDs. No credentials appear
in user-facing payloads.

These message semantics are breaking changes. Use session.v2 and versioned
runtime/HTTP contracts where needed, following the workspace contract policy.
Generate code and publish a proto tag before consumers pin it.

## Persistence

Use transaction functions through PostgREST /rpc. Revoke default public execution
and grant intended service roles only. Prefer invoker-security functions and
explicit schema references. Mutations lock the same session row and check all
preconditions. Add equivalent memory-store semantics for tests.

Use timestamped migrations. Coordinate pre-launch cutover across services and
frontend rather than maintaining a permanent compatibility projection.
If existing development histories matter, perform an explicit one-time import
marked as legacy. Previously lost tool ordering cannot be reconstructed exactly.
A database reset is a separately approved operational choice.

## Delivery and verification

1. Finalize canonical schema, provider adapters, and v2 contracts.
2. Implement storage, short transactions, root execution state, idempotency, leases, and tests.
3. Integrate runtime incremental persistence, hydration, and tool recovery.
4. Update backend/frontend structured history and streaming behavior, immutable
   template creation/cloning/archival, and frozen session configuration.
5. Implement token budgets, summaries, checkpoints, and telemetry.
6. Exercise multi-replica failure cases and enable compaction on canary sessions.

Acceptance tests must cover repeated compaction without gaps, unchanged history,
parallel tools and original IDs, restart equivalence, crashes around tool dispatch,
lost acknowledgements, conflicting writers, lease expiration, end/delete races,
oversized input, failed summaries, and unauthorized access. Verify interrupted
turn display and that persistence failure never emits a successful done event.
Verify root-only lifecycle fields, same-session root references, immutable
completed content, one active request per session, and atomic final output/status
updates. Verify that no database transaction remains open while a model or tool
call is pending.
Test mid-request compaction across repeated tool cycles, parallel result batches,
restart at safe boundaries, and cancellation racing checkpoint publication.
Verify busy sessions reject new requests while duplicate IDs reconnect and
cancellation remains available. Test archived-template admission, immutable fields,
cloning, existing-session continuity, and frozen platform configuration.

Use Node built-in tests, Go stdlib tests, required build/vet checks, and buf
generation/lint/drift checks. Test transactions, privileges, and locks against
local Postgres/Supabase; mocked HTTP cannot prove them. Validate frontend flows
using available browser tooling without adding an unrequested test framework.

Record token reduction, compaction duration/usage, IDs, revisions, lease conflicts,
write latency, and hydration failures. Never log bodies or credentials.
Measure added write cost from incremental persistence.

Disabling automatic compaction must retain checkpoint-aware hydration. Rollback
requires a canonical-message/checkpoint-aware binary. Original messages remain
available throughout; cold storage is not required for this design.

## Related decisions

- [Immutable Agent Templates](decisions.md#d001-immutable-agent-templates)
