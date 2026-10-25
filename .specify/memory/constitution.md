# Social OS Constitution

<!-- Canonical location: this file is what /speckit-* commands read as the project
     constitution. Human-readable copy: docs/CONSTITUTION.md (keep in sync, edit here first).
     Derived from "Social Media OS Development.docx". -->

## Core Principles

### I. Self-hosted and Docker-first (P1)
Everything runs on a single capable workstation through `docker compose up`. No component
may require a cloud-hosted service to function in development or production. External SaaS
is permitted only as an optional, configured integration. **Verify:** a clean clone plus
`.env` and `docker compose up -d` yields a working stack with no manual step outside the
compose file.

### II. Local AI by default, CPU-compatible (P2)
AI runs on CPU-only hardware with small quantized models (Ollama) plus an embedding model.
GPU or hosted inference is an upgrade, never a requirement. No feature hard-codes a cloud
provider; every AI feature degrades to a local model path.

### III. Canonical model, adapters at the edge (P3)
Core domain (`SocialAccount`, `Content`, `PlatformPost`, `Engagement`) never contains
platform-specific logic. Platform differences live behind `SocialPlatformAdapter`
implementations exposing an explicit capabilities matrix (`publish_text`, `publish_image`,
`publish_video`, `read_comments`, `reply_comments`, `read_analytics`). Adding a platform
means a new adapter module and registration only. Unsupported capabilities are refused.

### IV. Official APIs first, automation as fallback (P4)
Publishing and engagement use official platform APIs. Playwright/browser automation is a
fallback adapter and a testing tool only; it must never be the foundation of an
integration, and must respect each platform's terms.

### V. FastAPI is the system of record; n8n is glue (P5)
Business logic, auth, accounts, content, campaigns, scheduling, analytics, agents,
permissions, adapters and all database writes live in FastAPI. n8n performs scheduled
workflows, triggers, notifications and external automation only. n8n never owns domain
data and calls API endpoints instead of touching the database.

### VI. Provider-agnostic AI layer (P6)
All model access goes through one `LLMProvider` interface (`generate`, `structured_output`,
`embed`). Ollama, OpenAI, Anthropic and vLLM are interchangeable implementations selected
by configuration. No module outside the provider package imports a vendor SDK.

### VII. Agent least privilege (NON-NEGOTIABLE) (P7)
Every agent declares CAN / CANNOT permissions (research cannot publish; publishing cannot
rewrite strategy or delete accounts). Enforcement is in code, not prompts, and is
unit-tested per agent. An unauthorized action returns a denial, not a model refusal.

### VIII. Human approval is the default (P8)
Autonomous action is opt-in through a policy engine, per risk class. Publishing and
engagement require approval until policy explicitly relaxes them. Low-risk replies may be
auto-published; politics, legal accusations, complaints and controversy are human-only;
abuse is ignored. With default policy, no content reaches a platform without a recorded
approval event.

### IX. Secrets are encrypted and never in the tree (P9)
OAuth tokens and API keys are encrypted at rest (`encrypted_access_token`,
`encrypted_refresh_token`, `token_expires_at`, `scopes`). Secrets never appear in source,
logs, images, browser storage or client bundles; third-party keys live in the environment.
Repo scans find no credential material and DB dumps show ciphertext only.

### X. Auditability and the content state machine (P10)
Content moves only through `IDEA → RESEARCHED → DRAFT → AI_REVIEWED → HUMAN_REVIEW →
APPROVED → SCHEDULED → PUBLISHED → MONITORING → ANALYZED → LEARNING`. Every state change,
agent run, approval and publish attempt is recorded. Any published post can be
reconstructed — who approved it, which variant was sent, the exact payload.

### XI. Contract-first, API-first (P11)
`docs/API-CONTRACTS.md` plus the generated `openapi.json` are authoritative. Code changes
to match the contract, or the contract changes deliberately first. Frontend and backend
develop against the contract, never against each other; drift fails CI.

### XII. Testability is a requirement, not a phase (P12)
Every feature ships tests at the level it operates (unit, API, integration) and is
exercised the way a user exercises it. A green build is not evidence; recorded evidence is.
Acceptance criteria must be demonstrable by command.

### XIII. Spec-driven development is the only path to code (P13)
No feature is implemented before its spec is reviewed, planned, tasked and converged:
`constitution → specify → clarify → plan → tasks → implement → converge`. Every feature
folder contains `spec.md`, `plan.md`, `tasks.md` and `checklist.md`, with the checklist
fully evidenced before merge.

### XIV. Supported ≠ available (P14)
Architecture may support a capability long before it is operationally available.
Publishing, analytics and engagement depend on each platform's API permissions, account
type and app approval. Specs state which is which, the capabilities matrix reflects
reality, and no UI or doc claims a capability that is not a live, permitted path.

## Additional Constraints

- Stages over sprints of everything: build in the seven documented phases; Phase N+1
  capability work does not begin inside Phase N.
- Interfaces before implementations: Phase 1 defines extension points (adapter interface,
  publish pipeline stages, domain events, policy shape, agent tables) without filling them.
- Fail closed: unknown permissions, ambiguous intent or missing policy → deny and ask a
  human.
- Data: PostgreSQL + pgvector is the system of record and vector store; MinIO holds blobs
  with metadata only in Postgres; Redis holds queue/cache/rate limits.
- Reversibility: small steps, versioned migrations (expand → migrate → contract), pinned
  image tags and lockfiles.

## Development Workflow

- Spec-driven loop: `/speckit-constitution` → `/speckit-specify` → `/speckit-clarify` →
  `/speckit-plan` → `/speckit-checklist` → `/speckit-tasks` → `/speckit-analyze` →
  `/speckit-implement` → `/speckit-converge`.
- Artifacts live in `specs/<NNN-feature>/`; project documents live in `docs/`.
- Human checkpoint after each artifact: review before the next command runs.

## Quality Gates

A change is done only when all hold: (1) spec/plan updated or inside an approved task;
(2) contract, migrations and code consistent; (3) lint/typecheck/tests pass locally and in
CI; (4) acceptance criteria exercised with evidence recorded in `checklist.md`;
(5) no secret, credential or PII introduced; (6) Docker image builds and `docker compose
up` works from clean.

## Governance

This constitution supersedes all other practices in this repository. It outranks any
individual spec or plan: a violating spec or implementation must be changed, or the
constitution amended through a recorded decision. Amendments require a dated row in the
Amendments log, a one-paragraph justification, and review of every affected feature spec.
All plans and reviews must verify compliance with the gates above.

**Version**: 1.0 | **Ratified**: 2026-10-25 | **Last Amended**: 2026-10-25
