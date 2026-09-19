# Agent Instructions

Follow [Duke-ECE engineering standards](https://github.com/Duke-ECE/standards/blob/main/AGENTS.md).

- Write all documentation, examples, and comments in English.
- Organize work by iteration, not by service. Use a stable numeric ID and a descriptive slug.
- Each iteration starts with README.md, design.md, and delivery.md. Add files only when needed.
- Use Draft, Accepted, Implementing, Completed, Deferred, or Cancelled for iteration status.
- Keep agreed decisions distinct from unresolved details. Do not mark an entire design Accepted merely because individual decisions were approved.
- architecture/ describes verified, implemented behavior. Proposed behavior belongs in iterations/.
- Record durable cross-iteration decisions in decisions/ and link them from designs.
- Keep authoritative proto files, SQL migrations, and implementation code in their owning repositories.
- Link cross-repository PRs, release tags, verification results, and rollout/rollback evidence from delivery.md.
- Do not mark work complete without evidence. Never invent test results or deployment status.
- Preserve completed iteration paths; update indexes instead of moving documents into archive folders.
- Before publishing, check relative links, whitespace, and documents for secrets. No automated build is required.
