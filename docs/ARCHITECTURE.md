# Social OS — Technical Foundation (Phase 1)

**Status:** Draft · **Version:** 0.1 · **Date:** 2026-10-25
**Governs:** repository layout, services, data model, jobs, configuration, local development.
Companion docs: [CONSTITUTION.md](CONSTITUTION.md) · [PHASE-1-SPEC.md](PHASE-1-SPEC.md) ·
[API-CONTRACTS.md](API-CONTRACTS.md) · [ACCEPTANCE-CRITERIA.md](ACCEPTANCE-CRITERIA.md)

---

## 1. Repository layout

```
social-os/
├── apps/
│   ├── web/                      # Next.js + TypeScript (Social Command Center UI)
│   └── api/                      # FastAPI application package
│       ├── app/
│       │   ├── main.py           # app factory, middleware, router mount
│       │   ├── api/v1/           # route modules (one per resource)
│       │   ├── core/             # config, security, logging, errors
│       │   ├── domain/           # entities, state machine, policies, events
│       │   ├── services/         # use-case layer (no framework knowledge)
│       │   ├── repositories/     # persistence adapters (SQLAlchemy)
│       │   ├── adapters/         # SocialPlatformAdapter + DryRunAdapter
│       │   └── ai/               # LLMProvider interface (unused in Phase 1)
│       └── tests/
├── services/
│   ├── worker/                   # publish/notification job consumer
│   └── scheduler/                # cron-shaped entrypoint (research, digests later)
├── integrations/                 # one package per platform (Phase 2 fills these)
├── database/
│   ├── migrations/               # Alembic revisions
│   └── seed/                     # demo data
├── workflows/                    # n8n workflow exports (Phase 5)
├── docker/                       # Dockerfiles, entrypoints, nginx/caddy config
├── docs/                         # specs and SDD artifacts (this set)
├── .specify/                     # Spec Kit templates, commands, presets
├── docker-compose.yml
├── docker-compose.override.yml   # local dev only (Mailpit, hot reload)
└── .env.example
```

**Rules:** `apps/api/app/domain` imports nothing from FastAPI, SQLAlchemy or vendor SDKs.
`services/*` call `apps/api` services in-process or over HTTP — they never import route
modules. `integrations/*` are pure adapters registered at startup.

## 2. Services (docker-compose)

| Service | Image / build | Ports | Role |
|---|---|---|---|
| `web` | build `apps/web` | 3000 | Next.js UI |
| `api` | build `apps/api` | 8000 | FastAPI, OpenAPI at `/docs` |
| `worker` | build `services/worker` | — | consumes job queue (publish, notify) |
| `scheduler` | build `services/scheduler` | — | cron entrypoint; Phase 5 delegates to n8n |
| `db` | `postgres:16` + pgvector | 5432 | system of record + vectors |
| `redis` | `redis:7` | 6379 | queue, cache, rate limits |
| `minio` | `minio/minio` | 9000/9001 | object storage for media |
| `mailpit` | `axllent/mailpit` | 8025 | dev-only SMTP capture |
| `caddy` | `caddy:2` | 80/443 | reverse proxy (prod profile) |
| `n8n`, `ollama`, `prometheus`, `grafana`, `loki` | profiles `phase5`, `ai`, `obs` | — | not started in Phase 1 |

Compose conventions: named volumes for `pgdata`, `minio-data`, `redis-data`; healthchecks
on every service; `depends_on: condition: service_healthy`; all images pinned by digest or
minor tag; `restart: unless-stopped`; no secrets in the compose file.

## 3. Configuration

- `apps/api/app/core/config.py` reads environment only (pydantic-settings); no config
  files at runtime beyond `.env`.
- `.env.example` documents every variable; `.env` is git-ignored (P9).
- Namespaces: `APP_*`, `DATABASE_URL`, `REDIS_URL`, `S3_*`, `JWT_*`, `ENCRYPTION_KEY`,
  `SMTP_*`, `PUBLISH_*`, `LLM_*`, `POLICY_*`.
- `ENCRYPTION_KEY` is required for boot in production; dev generates one and stores it in
  `.env`.
- Reserved for later phases: `OPENAI_API_KEY` (never committed), `N8N_*`, `OLLAMA_*`.

## 4. Data model

Entities (all prefixed `smos_`): the full 19-table set from the design document is created
in Phase 1 migrations so later phases add columns, not tables. Two adjustments from the
clarified Phase 1 spec: `scheduled_posts` + `published_posts` are merged into a single
`platform_posts` (one independently managed post per target account), and Phase 1 adds
`auth_attempts` (brute-force counters) and a durable `jobs` outbox. See
`specs/001-phase-1-command-center/data-model.md`.

| Table | Phase 1 use |
|---|---|
| `users`, `workspaces`, `memberships`, `invitations` | auth & RBAC |
| `auth_attempts` | brute-force counters and lockouts (rolling window, per identity and per IP) |
| `social_accounts`, `platform_tokens` | registry (token exchange in Phase 2) |
| `campaigns`, `content_items`, `content_variants` | content domain |
| `media_assets` | MinIO metadata only (no blobs in Postgres, P9) |
| `platform_posts` | one row per target account: schedule, dispatch, retry, outcome (merges `scheduled_posts` + `published_posts`) |
| `comments`, `engagements` | created empty; Phase 2/6 fill |
| `analytics_snapshots` | created empty; Phase 2/7 fill |
| `agent_runs`, `agent_tasks`, `agent_memories` | created empty; Phase 4 fills |
| `approval_requests`, `automation_rules`, `audit_events` | approvals, policy, audit |

Key columns and relations:

```
workspaces 1—* memberships *—1 users
workspaces 1—* social_accounts 1—1 platform_tokens
workspaces 1—* campaigns 1—* content_items 1—* content_variants
content_items 1—* media_assets (M:N via content_variant_media)
content_items 1—* approval_requests
content_variants 1—* platform_posts *—1 social_accounts
platform_posts 1—* jobs (one per attempt group)
platform_posts 1—* engagements (comments, later)
* → audit_events (polymorphic: entity_type, entity_id, actor_type, actor_id)
```

**Content lifecycle** (`content_items.status`) is a DB-level CHECK constraint plus
application-level transition table:

```
IDEA → RESEARCHED → DRAFT → AI_REVIEWED → HUMAN_REVIEW → APPROVED → SCHEDULED
     → PUBLISHED → MONITORING → ANALYZED → LEARNING
                     ↘ CHANGES_REQUESTED → DRAFT      ↘ FAILED → HUMAN_REVIEW
```

**Money/time conventions:** `timestamptz` in UTC, ISO-8601 at the API boundary; soft
deletion via `deleted_at`; every table has `id (uuid)`, `workspace_id`, `created_at`,
`updated_at`.

**Timezone:** one IANA timezone per workspace (`workspaces.timezone`, validated on
write). Instants are stored and dispatched in UTC; the workspace timezone is a display
and input concern only — changing it never reschedules an existing post (Phase 1 spec
FR-026).

**Tokens:** `platform_tokens(encrypted_access_token, encrypted_refresh_token,
token_expires_at, scopes, last_refreshed_at)` — AES-GCM via `ENCRYPTION_KEY` (P9).

**Vectors:** pgvector columns on `agent_memories.embedding` and
`content_items.embedding` (nullable, unused in Phase 1).

## 5. Background jobs

Queue: Redis (list + `BRPOP`) with a Postgres `jobs` outbox table for durability; the
worker claims with `FOR UPDATE SKIP LOCKED` so a crash never loses a job.

| Job | Trigger | Behaviour |
|---|---|---|
| `publish_post` | scheduler at `scheduled_at - 0s` | validate → format → dispatch adapter → record → emit `post.published` |
| `notify` | approval created, publish outcome | in-app notification + email via Mailpit/SMTP |
| `reconcile_schedule` | on worker boot | re-enqueue future posts missed during downtime |
| `retry` | exponential backoff 5s/30s/2m/10m, max 4 attempts | then `FAILED` + operator notification |
| `purge_retention` | daily | soft-deleted content > 30 d, notifications > 90 d, `auth_attempts` > 24 h — and nothing else |

Idempotency: `jobs.idempotency_key = (platform_post_id, attempt_group)`; duplicate
dispatch is impossible. The scheduler (`APScheduler`/cron) only *enqueues*; only the
worker *executes*.

## 6. Migrations and lifecycle

- Alembic, one revision per change, autogenerated then reviewed.
- `alembic upgrade head` runs automatically on `api` boot in dev; in production it is an
  explicit, logged step before rollout.
- Rollback: each revision implements `downgrade()`; destructive changes are two-release
  (expand → migrate → contract).
- `make db.reset` = drop, create, upgrade, seed.

## 7. Logging, health, errors

- Structured JSON logs: `ts`, `level`, `request_id`, `route`, `workspace_id`, `actor_id`,
  `duration_ms`. Never log tokens, bodies of private content, or secrets (P9).
- `GET /healthz` (process alive), `GET /readyz` (db + redis + minio reachable).
- Errors: single envelope `{error: {code, message, details, request_id}}`; domain errors
  map to 4xx, unexpected to 500 with no internals exposed.

## 8. Local development workflow

```bash
git clone <repo> && cd social-os
cp .env.example .env          # generates ENCRYPTION_KEY
docker compose up -d           # db, redis, minio, api, worker, scheduler, web, mailpit
make db.reset                 # migrations + seed
make test                     # unit + API tests
open http://localhost:3000    # UI
open http://localhost:8000/docs
open http://localhost:8025    # Mailpit
```

Hot reload: `api` and `web` mount source volumes in `docker-compose.override.yml`.
`make check` = lint + typecheck + tests (the same target CI runs).

## 9. Security baseline (Phase 1)

- Argon2id password hashing; short-lived session cookie, `HttpOnly`/`SameSite=Lax`,
  rotation on privilege change; CSRF token for cookie-authenticated mutations.
- Failed sign-ins: 10 failures per account in a rolling 15-minute window locks the
  account for 15 minutes; counters live in Postgres (`auth_attempts`) so a cache flush
  cannot unlock an attacker, IP throttling is independent, and every auth error returns
  the same generic message (P10 of the clarified spec).
- RBAC checked in the service layer, never only in the UI (P7 pattern).
- Every query scoped by `workspace_id` from the session — never from request bodies.
- Upload limits, MIME allow-list, content-derived filenames; blobs served via short-lived
  signed URLs.
- Rate limiting per IP and per session (Redis).
