# Implementation Plan: Social Command Center (Phase 1)

**Branch**: `001-phase-1-command-center` | **Date**: 2026-10-26 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-phase-1-command-center/spec.md`

## Summary

Build the whole Social Command Center — identity and RBAC, social account registry with
capability matrices, content library with per-platform variants, media, campaigns,
calendar, the content lifecycle with human approval, conflict-checked scheduling with
fan-out to several accounts, a durable at-most-once publish pipeline through a dry-run
adapter, notifications, audit and a backup/restore procedure — on Docker Compose with no
AI and no third-party platform calls.

Technical approach (from [research.md](research.md)): one canonical `platform_posts`
record per target account replaces the baseline's `scheduled_posts`/`published_posts`
pair; scheduling and conflict checks run inside a single Postgres transaction that
serialises per social account; publication is driven by a durable outbox claimed with
`FOR UPDATE SKIP LOCKED` and an idempotency key, so a crash can neither lose nor double
fire a post; brute-force counters live in Postgres so a cache flush cannot unlock an
attacker; all times are UTC instants displayed in one IANA workspace timezone.

## Technical Context

**Language/Version**: Python 3.12 (API, worker, scheduler), TypeScript 5.x on
Next.js 15 (web)

**Primary Dependencies**: FastAPI, SQLAlchemy 2, Alembic, Pydantic v2, Argon2id
(passwords), httpx; Next.js/React; adapters behind a `Protocol` (stdlib only in Phase 1)

**Storage**: PostgreSQL 16 + pgvector (system of record), Redis 7 (job queue, cache),
MinIO (media blobs), all on named volumes

**Testing**: pytest + httpx ASGI transport + factory fixtures (API/unit), a fake clock
for scheduling, Vitest + Playwright (web); `make check` = lint + typecheck + tests

**Target Platform**: Linux/amd64 and arm64 containers via Docker Compose on a single
workstation; no cloud dependency (P1)

**Project Type**: web service + web application (monorepo, `apps/api` + `apps/web` +
`services/*`)

**Performance Goals**: list endpoints p95 < 300 ms, dashboard p95 < 500 ms and < 1 s
cold on a seeded database, scheduled post dispatched within 15 s of `scheduled_at`

**Constraints**: runs offline from a clean clone; no secrets in the tree (P9); tokens
encrypted at rest; no platform API calls (dry-run adapter only); approval required
before scheduling by default (P8); 15-minute lockout after 10 failures; spacing window
default 60 min; retention fixed (audit/content forever, soft-delete 30 d, notifications
90 d)

**Scale/Scope**: 3 roles × 26 functional requirements, 13 user stories, ~45 endpoints,
20 tables (19 baseline with `scheduled_posts`/`published_posts` merged into
`platform_posts`, plus `auth_attempts` and `jobs`), 19 alembic-managed migrations
target, single-workspace-team usage (tens of users, ~10k content items)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design. Source:
`.specify/memory/constitution.md`.*

| # | Gate | Pass? | Evidence in this plan |
|---|---|---|---|
| P1 | Runs entirely via `docker compose up`, no cloud dependency | PASS | Compose services in §2 of ARCHITECTURE; quickstart scenario 1 boots offline; no hosted service in any dependency above |
| P2 | Local/CPU AI path intact (or `LLMProvider` interface untouched) | PASS | `AI_REVIEWED` traversed without a model; `apps/api/app/ai/` created with the interface and no provider wired (spec FR-009, research R8) |
| P3 | No platform-specific logic in core domain; capabilities matrix respected | PASS | Only `DryRunAdapter` implements the Protocol; capability matrix stored as data on `social_accounts`; `422` on unsupported capability (research R6) |
| P4 | No browser automation used where an official API path exists | PASS | No Playwright adapters; Playwright appears only as a browser test tool, never as an adapter |
| P5 | Business logic in FastAPI; automation only calls API endpoints | PASS | Lifecycle, approval, spacing and lockout rules live in `app/services`; worker/scheduler call the same services, not route handlers |
| P6 | Model access only through `LLMProvider` | PASS | No model call anywhere in this feature; interface present, unwired |
| P7 | Agent/actor permissions enforced in code for this change | PASS | Role checks in the service layer for every mutating use case; SC-007 authorization matrix test |
| P8 | Approval/policy defaults unchanged or deliberately relaxed with justification | PASS | Approval required before scheduling remains the default; only an owner with a reason can override a spacing conflict (FR-023), audited |
| P9 | No secrets introduced; tokens encrypted at rest | PASS | `.env`-only configuration, git-ignored; `platform_tokens` AES-GCM columns; backup excludes plaintext secrets (FR-025) |
| P10 | State transitions and audit events cover this change | PASS | Transition table enforced with `409` + allowed list; every mutation writes `audit_events`; SC-004/SC-010/SC-011 evidence |
| P11 | Contract (`docs/API-CONTRACTS.md`, `openapi.json`) updated first | PASS | Contract reconciled to `platform_posts` in this step before any code; deltas staged in [contracts/api.md](contracts/api.md) |
| P12 | Tests and evidence defined for every requirement here | PASS | Every FR maps to a scenario/SC; tasks.md will require `make check` output plus checklist evidence |
| P13 | Spec → plan → tasks → converge respected for this feature | PASS | This plan follows spec.md (clarified 2026-10-26); tasks and converge come next |
| P14 | Capabilities marked `available` vs `supported-only` accurately | PASS | Phase Gate table in spec.md; dry-run marked `dry_run=true` in every publish record and event |

**Result**: PASS (continue) — no gate failed, so Complexity Tracking stays empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-phase-1-command-center/
├── plan.md              # This file (/speckit-plan output)
├── research.md          # Phase 0 output — R1…R8 decisions
├── data-model.md        # Phase 1 output — entities, fields, transitions
├── quickstart.md        # Phase 1 output — runnable validation scenarios
├── contracts/
│   ├── api.md           # Phase 1 output — endpoint contract for this feature
│   └── events.md        # Phase 1 output — domain events + adapter payload
├── checklists/
│   └── requirements.md  # Spec quality checklist (already passing)
├── spec.md              # Feature specification (clarified)
└── tasks.md             # Phase 2 output — /speckit-tasks, NOT created here
```

### Source Code (repository root)

```text
apps/web/                      # Next.js UI: dashboard, calendar, content, settings
apps/api/app/
├── api/v1/                    # route modules: auth, accounts, content, media,
│                              #   campaigns, calendar, platform_posts, approvals,
│                              #   notifications, audit, settings, dashboard, health
├── core/                      # config, security (sessions/CSRF/lockout), logging, errors
├── domain/                    # state machine, policies, entities, domain events
├── services/                  # use cases: scheduling, spacing, publishing, retention
├── repositories/              # SQLAlchemy persistence
├── adapters/                  # SocialPlatformAdapter Protocol + DryRunAdapter
└── ai/                        # LLMProvider interface (unwired in Phase 1)
apps/api/tests/                # unit + API tests (fake clock, factories)
services/worker/               # job consumer: publish_post, notify, purge_retention
services/scheduler/            # cron entrypoint: enqueue due posts, reconcile on boot
integrations/                  # empty in Phase 1 (Phase 2 adds platforms)
database/migrations/           # Alembic revisions
database/seed/                 # demo data (FR-020)
scripts/backup.sh              # FR-025 backup + integrity check
scripts/restore.sh             # FR-025 restore into a clean environment
scripts/verify_restore.sh      # FR-025/SC-012 post-restore assertions
docker/, docker-compose.yml, docker-compose.override.yml, .env.example, Makefile
```

**Structure Decision**: the Social OS default layout from the template is adopted
without deviation — `apps/api` holds the API package with the domain layer isolated
from framework imports, `services/*` are thin job entrypoints over the same services,
and `integrations/*` stays empty until Phase 2. The only additions to the baseline
layout are `scripts/backup.sh`, `scripts/restore.sh` and `scripts/verify_restore.sh`,
which FR-025 requires as a documented, tested procedure.

## Complexity Tracking

> Fill ONLY if Constitution Check has violations that must be justified.

None — all 14 gates pass. Two design deviations from the baseline docs are recorded
rather than justified as violations, because neither weakens a gate:

| Deviation | Why | Where handled |
|---|---|---|
| `scheduled_posts` + `published_posts` merged into `platform_posts` | The clarified fan-out decision makes one independently managed post per account the unit of scheduling, approval, retry and analytics; two tables would split one lifecycle | [data-model.md](data-model.md), [contracts/api.md](contracts/api.md), and the contract-first edits to `docs/API-CONTRACTS.md` / `docs/ARCHITECTURE.md` |
| `auth_attempts` added as a new table | FR-001 requires lockout counters that survive a restart and cannot be bypassed by flushing a cache | [data-model.md](data-model.md), research R4 |
