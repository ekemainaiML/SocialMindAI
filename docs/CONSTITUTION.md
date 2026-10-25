# Social OS — Constitution

**Version 1.0 · Ratified 2026-10-25 · Status: Active**

The canonical constitution lives in [`.specify/memory/constitution.md`](../.specify/memory/constitution.md) —
that is the file Spec Kit's `/speckit-*` commands read as the project's highest authority.
**Edit it there first**, then mirror the change into this file.

This document is the human entry point: it indexes the principles and states how they are
enforced. Reference: [Social Media OS Development](../Social%20Media%20OS%20Development.docx).

---

## Why a constitution

The design document is a vision; the constitution is the part of it that may **not** be
negotiated by an individual spec, plan or implementation. A violating artifact is changed,
or the constitution is amended through a recorded decision — nothing is quietly overridden.

## Principle index

| # | Principle | One-line rule | Verification |
|---|---|---|---|
| P1 | Self-hosted, Docker-first | Clean clone + `.env` + `docker compose up -d` is the whole install | smoke sequence in [ACCEPTANCE-CRITERIA.md](ACCEPTANCE-CRITERIA.md) AC-1 |
| P2 | Local AI, CPU-compatible | AI works on CPU via Ollama; no hard-coded cloud provider | provider chosen by config only |
| P3 | Canonical model, adapters at the edge | Core domain has zero platform-specific logic; capabilities matrix is explicit | new platform = new adapter + registration |
| P4 | Official APIs first | Playwright is a fallback adapter, never the foundation | adapter dependency review |
| P5 | FastAPI is the system of record | n8n only automates; it never owns domain data or writes the DB | no business rule solely in a workflow |
| P6 | Provider-agnostic AI | One `LLMProvider` interface for generate/structured_output/embed | no vendor SDK outside the provider package |
| P7 | Agent least privilege (NON-NEGOTIABLE) | CAN/CANNOT enforced in code, not prompts | per-agent denial tests |
| P8 | Human approval by default | Policy engine gates publishing and engagement by risk class | no publish without an approval event |
| P9 | Secrets encrypted, never in the tree | Tokens encrypted at rest; nothing secret in source, logs or bundles | secret scan + ciphertext-only dump |
| P10 | Auditability + state machine | Content moves only through the defined states; every change audited | published post fully reconstructable |
| P11 | Contract-first, API-first | `docs/API-CONTRACTS.md` + `openapi.json` are authoritative | contract drift fails CI |
| P12 | Testability is a requirement | Tests ship with features; evidence, not a green build, proves done | ACs demonstrable by command |
| P13 | Spec-driven development only | No code before spec → plan → tasks → converge | every feature folder has all four artifacts |
| P14 | Supported ≠ available | Capability claims trace to a live permitted path or are marked "planned" | capabilities matrix matches reality |

## Quality bar (definition of done)

1. Spec and plan updated, or the change sits inside an approved task.
2. Contract, migrations and code are consistent.
3. Lint, typecheck and tests pass locally and in CI.
4. Acceptance criteria exercised, evidence recorded in the feature `checklist.md`.
5. No secret, credential or PII introduced into the repository or logs.
6. The Docker image builds and `docker compose up` still works from a clean clone.

## Amendments

Amending the constitution requires: editing `.specify/memory/constitution.md` first, a row
in its Amendments log with date and justification, mirroring here, and reviewing every
feature spec affected.

| # | Date | Principle | Change | Reason |
|---|------|-----------|--------|--------|
| 1 | 2026-10-25 | all | Initial constitution | Project inception from the design document |
