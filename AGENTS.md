# Agent Instructions

Follow [Duke-ECE engineering standards](https://github.com/Duke-ECE/standards/blob/main/AGENTS.md).

- Write all documentation, examples, and comments in English.
- Use one directory per requirement under requirements/, regardless of the number of affected repositories.
- Copy the root-level template/ directory in full to requirements/NNNN-descriptive-slug/.
- Each requirement contains README.md, design.md, decisions.md, and delivery.md. Add files only when needed.
- Update requirements/README.md when adding a requirement.
- Use Draft, Accepted, Implementing, Completed, Deferred, or Cancelled for requirement status.
- Keep agreed decisions distinct from unresolved details; individual accepted decisions do not imply full design approval.
- Append decisions to the requirement's decisions.md with local IDs such as D001.
- To replace an accepted decision, append a new entry and mark the old one Superseded with links. Proposed entries and editorial corrections may be revised.
- Cross-requirement decisions remain in their originating directory; link to them instead of duplicating them.
- architecture/ describes verified implemented behavior; proposals belong under requirements/.
- Authoritative proto files, SQL migrations, and implementation code remain in their owning repositories.
- Link affected repositories, PRs, release tags, verification, and rollout/rollback evidence from delivery.md.
- Do not invent test results or mark work complete without evidence.
- Preserve completed requirement paths; update the index instead of relocating completed work.
- Check relative links, whitespace, and secrets before publishing. No build is required.
