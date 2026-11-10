# Decision Log

## D001: Share identity and backend authorization across clients

Status: Accepted for implementation

Date: 2026-11-07

### Context

The web console uses Supabase Auth and the backend owns business APIs and access checks.

### Decision

Use the same Supabase Auth project in mobile, send its JWT to the backend, and keep business data behind backend APIs.

### Consequences

Mobile and web share accounts and server-side permissions. A mobile client still needs the browser-safe Supabase URL and publishable key, while service-role and LLM provider secrets stay server-side.

## D002: Use a native callback for GitHub login

Status: Accepted for implementation

Date: 2026-11-07

### Context

Expo Go cannot register an app-specific native URL scheme.

### Decision

Register `managedagents://auth/callback` in the app and Supabase redirect configuration. Test GitHub login in a native development build.

### Consequences

Email/password login remains testable in Expo Go. The native GitHub path requires applying the Supabase redirect configuration before device acceptance.
