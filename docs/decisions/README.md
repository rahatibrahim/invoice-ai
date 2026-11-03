# Decisions — invoice-ai

Architecture decision records (ADRs) for significant technical choices. One file per decision, named `NNNN-short-title.md` (e.g. `0003-extraction-approach.md`). Once a decision is accepted, its record is not rewritten; a later decision that changes it gets a new record that supersedes the old one.

## Index

| ADR | Decision | Status |
|---|---|---|
| 0001 | Python as the pipeline language | Proposed |
| 0002 | LLM reads; deterministic code normalizes and validates | Proposed |
| 0003 | Extraction approach (A / B / C) and AI provider | Pending spike |
| 0004 | PDF libraries (pdfplumber + pypdfium2, avoid AGPL) | Proposed |
| 0005 | Release 2 web framework | Pending |

The list matches `02-architecture.md` §9. Records are written when a decision is confirmed.

## Template

```markdown
# NNNN — Title

| Item | Value |
|---|---|
| Status | Proposed / Accepted / Superseded by NNNN |
| Date | YYYY-MM-DD |

## Context
What problem or question this decision answers, and the constraints.

## Options
The options considered, with their pros and cons. Include numbers (accuracy, cost, time) where available.

## Decision
What was chosen.

## Consequences
What this makes easier or harder, and what follows from it.
```
