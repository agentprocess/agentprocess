# Notes for agents working in this repository

Agent Process is a small, server-neutral protocol. Keep it small: new requirements impose costs on every implementation and must answer a demonstrated need from a written process, not hypothetical completeness. When a trade-off is open, the simpler and more adoptable choice wins.

- `spec/specification.md` and `spec/tools.md` are authoritative, with the profiles in `spec/profiles/`. Guides, examples, the skill, schemas and implementations add no requirements. Where they disagree with the specification, they are wrong.
- The tool contracts are exact. Prose elsewhere may be plain; a tool's arguments, results and errors may not be ambiguous.
- `schemas/core-2/` is generated from the reference implementation's contracts. Do not edit it by hand; change `spec/tools.md` and regenerate.
- Every change to the specification gets an entry in `spec/changelog.md` saying what changed and what prompted it.
- No product names in the specification, profiles or skill. A server's own features belong in that server's own profile and documentation.
- Write plainly: short sentences, tables for fields, one example per shape.
