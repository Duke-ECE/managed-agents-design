# Technical Design

Status: Accepted

## Problem and goals

DeepSeek Flash enables high-effort thinking by default. PolyAgent currently
forces it off to reduce time to first visible answer, but users need to choose
the quality/latency tradeoff. The choice is made at template or session
creation, then optionally disable it for one later message.

## Decisions and alternatives

See [D001-D004](decisions.md). Session configuration remains immutable.

## Data model and contracts

`thinking_mode` is one of `off`, `low`, `high`, `max`, or
`provider_default`. A missing value means inherit, not provider default.
Resolution at session admission is: explicit session value, template value,
then the compatibility default. That default is `off` for official DeepSeek
Flash and `provider_default` for all other models.

`POST /api/agents` and agent reads carry optional `thinking_mode`. The stored
template value is nullable so omission remains distinguishable from explicit
`provider_default`. `POST /api/sessions` accepts optional top-level
`thinking_mode` alongside `agent_id`; a user-issued API key may specify it
while still being unable to override the template's LLM triple. The resolved
mode is returned on session creation and in an owner-scoped safe config read.
`POST /api/sessions/:id/messages` accepts an optional `thinking_mode: "off"`
alongside `content`. An omitted field uses the frozen session setting. Any
other per-message value is rejected with HTTP 400.

The resolved mode is frozen on the durable session record at admission, before
any chat turn. An additive `session.v1` field carries it through
CreateSession/GetSession and an additive `runtime.v1.LlmConfig` field carries
it to the runtime. The runtime includes it in the legacy `config` turn,
using the durable record's value on a pre-first-turn resume. Add it to the
`session.v2.SessionConfig` snapshot and configuration hash for the v2 cutover.
Older records and transcript turns with no mode retain their prior behavior;
template changes never alter an existing session.

For `deepseek-flash` on `api.deepseek.com`, `off` sends
`thinking: {"type":"disabled"}`. `low`, `high`, and `max` send
`thinking: {"type":"enabled"}` plus matching `reasoning_effort`.
`provider_default` omits both parameters. Explicit `off/low/high/max` on
other model/endpoints is rejected with HTTP 400 before creating a durable
record. This release does not stream reasoning content to the client. Provider
reasoning needed for tool-call continuity must remain server-private in the
durable transcript and be restored in message order; the owner transcript
read path must redact it.

## Execution and recovery

`polyagent chat --agent <id> --thinking <mode> <message>` sets the mode for a
new session. `polyagent chat --session <id> --thinking off <message>` disables
thinking for that one turn; an unflagged continuation uses the frozen mode.
Agent creation JSON accepts `thinking_mode`,
providing a reusable preference. The web client may omit the field; its
template form must preserve a value when editing or cloning.

Validate mode and model support in the backend before session-manager creates
a durable row. The runtime validates again so direct gRPC callers cannot
silently bypass it. Both user JWT and user-issued API key session creation
may set the mode, subject to the existing agent/LLM authorization rules.

## Verification

- A fake provider inspects exact outgoing JSON for all five modes.
- API tests cover precedence, unsupported-mode rejection before durable
  creation, user/API-key authorization, and legacy defaults.
- Runtime restart before the first chat turn and after a completed turn
  preserves the selected mode; template edits cannot change it.
- A thinking-enabled tool call survives restart with its full reasoning
  content in the next provider request, while owner transcript reads omit it.
- CLI tests cover new-session flags, continued sessions, and invalid modes.
- Live evaluation compares identical prompts at `off`, `low`, and `high`,
  recording session creation, first visible answer, total latency, and quality.

## Open questions

None for the first DeepSeek Flash release.
