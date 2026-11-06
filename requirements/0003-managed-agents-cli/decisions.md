# Decision Log

## D001: Go CLI with browser login

Status: Accepted

Date: 2026-11-06

### Context

The user requested a GitHub/Vercel-style CLI, implemented in Go, with browser login and account, agent, and session management.

### Decision

Create `Duke-ECE/managed-agents-cli`. Use Supabase GitHub OAuth with PKCE and the existing allow-listed `http://localhost:5173` loopback callback, plus email/password fallback. Use user JWTs for management APIs and keep third-party API keys separate.

### Consequences

Browser login cannot run while another process owns port 5173. No production Supabase redirect configuration change is needed for this first callback. Tokens are never printed by the CLI.

### Related work

- [Technical design](design.md)
- [Third-party chat SDK](../0002-third-party-chat-sdk/README.md)
