---
description: Conventional commit-message rules applied across the repository.
applyTo: "**"
---
- Write commit subjects in the imperative mood: "Add", "Fix", "Refactor" -- not
  "Added" or "Fixes".
- Keep the subject line at or under 50 characters; wrap the body at 72.
- Prefix the subject with a Conventional Commit type: `feat:`, `fix:`,
  `docs:`, `refactor:`, `test:`, `chore:`.
- Explain *why* in the body, not *what* -- the diff already shows what changed.
- Reference the issue or ticket id in the footer (e.g. `Refs: #123`).
- One logical change per commit; do not bundle unrelated edits.
