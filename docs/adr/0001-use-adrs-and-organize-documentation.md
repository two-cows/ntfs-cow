# ADR 0001: Use ADRs and organize documentation

Date: 2026-09-18

Status: Accepted

## Context

The project needs to preserve the reasoning behind significant architectural and project decisions. The process should require little maintenance for a small group of contributors.

## Alternatives considered

Keeping decisions only in conversations would leave their rationale outside the repository. A larger documentation framework would introduce maintenance before a concrete need exists.

## Decision

Use lightweight architecture decision records (ADRs), following [Michael Nygard's approach](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions).

- Store records in `docs/adr/NNNN-short-title.md`. Assign sequential numbers and never reuse them.
- Use `docs/README.md` as the documentation entry point and ADR index, listing each record's number, linked title, and status.
- Include a title, date, status, context, material alternatives considered, decision, and consequences. Keep the detail proportional to the decision.
- Use Proposed while discussing a decision, Accepted when agreed, and Rejected when declined.
- When a later decision changes an accepted one, create a new ADR, mark the earlier record Superseded, and link both records to each other. Preserve the earlier reasoning. Make editorial corrections in the existing record.
- Keep learning notes and general educational material out of the repository. When understanding code requires knowledge of a specific mechanism, link directly to relevant documentation from the code.

Maintain the Markdown records and index directly. A separate template, generator, or approval workflow is unnecessary at the current scale.

## Consequences

Contributors can find the rationale and history of project choices. Authors must keep the index, statuses, and links consistent as decisions change.
