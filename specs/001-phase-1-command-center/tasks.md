---

description: "Task list for Social Command Center (Phase 1)"
---

# Tasks: Social Command Center (Phase 1)

**Input**: Design documents from `/specs/001-phase-1-command-center/`

**Prerequisites**: [plan.md](plan.md) (required), [spec.md](spec.md) (user stories),
[research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/),
[quickstart.md](quickstart.md)

**Tests**: MANDATORY in this project (Constitution P12). Every user story carries
contract/API and integration tests that are written first and observed failing, plus an
evidence task that records the acceptance-criteria run in
`checklists/requirements.md`.

**Organization**: Tasks are grouped by user story so each story can be implemented,
tested and delivered independently.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US3, US4, …)
- Every task names its exact file path.

## Path Conventions (Social OS)

- **UI**: `apps/web/` (Next.js)
- **API**: `apps/api/app/{api/v1,core,domain,services,repositories,adapters,ai}/`
- **Tests**: `apps/api/tests/{unit,api,integration}/`, UI tests in `apps/web/`
- **Jobs**: `services/worker/`, `services/scheduler/`
- **Platform adapters**: `integrations/<platform>/` (empty in Phase 1)
- **Migrations**: `database/migrations/`
- **Scripts**: `scripts/`
- **Feature artifacts**: `specs/001-phase-1-command-center/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: project initialization — the Compose stack, the two application shells, the
tooling gate `make check`.

- [ ] T001 Create the repository skeleton directories from plan.md §Project Structure in
  `apps/web/`, `apps/api/app/{api/v1,core,domain,services,repositories,adapters,ai}/`,
  `apps/api/tests/{unit,api,integration}/`, `services/worker/`, `services/scheduler/`,
  `integrations/`, `database/{migrations,seed}/`, `docker/`, `scripts/`
- [ ] T002 [P] Pin the Python project in `apps/api/pyproject.toml` — Python 3.12,
  FastAPI, SQLAlchemy 2, Alembic, Pydantic v2, argon2-cffi, httpx, redis, boto3 (S3
  client for MinIO), pytest, ruff, mypy; versions resolved at install time, lockfile
  committed
- [ ] T003 [P] Pin the web project in `apps/web/package.json` — Next.js 15, React 19,
  TypeScript 5.x, ESLint, Vitest, Playwright; `bun install` (or npm) lockfile committed
- [ ] T004 [P] Write the image definitions `docker/docker/api.Dockerfile`,
  `docker/docker/worker.Dockerfile`, `docker/docker/scheduler.Dockerfile`,
  `docker/docker/web.Dockerfile` (multi-stage, non-root, no secrets baked in — P9)
- [ ] T005 Write `./docker-compose.yml` (and `./docker-compose.override.yml` for local
  hot reload and Mailpit) with `web`, `api`, `worker`, `scheduler`, `db` (postgres:16 +
  pgvector), `redis`, `minio`, `mailpit`, `caddy` per ARCHITECTURE §2 — named volumes,
  healthchecks, `depends_on: service_healthy`, images pinned, no secrets in the file
- [ ] T006 [P] Write `./.env.example` documenting every variable from ARCHITECTURE §3
  (`APP_*`, `DATABASE_URL`, `REDIS_URL`, `S3_*`, `ENCRYPTION_KEY`, `SMTP_*`, `PUBLISH_*`,
  `POLICY_*`) with placeholders only; confirm `.env` is git-ignored
- [ ] T007 [P] Write `./Makefile` targets `check`, `lint`, `typecheck`, `test`, `db.reset`,
  `db.upgrade`, `seed`, matching ARCHITECTURE §8; `check` = lint + typecheck + test and
  is the same target CI runs
- [ ] T008 [P] Write lint/format/typecheck configuration — `apps/api/ruff.toml` +
  `apps/api/mypy.ini`, `apps/web/.eslintrc` + `apps/web/tsconfig.json` — and
  `.github/workflows/ci.yml` that runs `make check`
- [ ] T009 Create the FastAPI application factory `apps/api/app/main.py` (middleware
  slots, router mount point, no business logic) and the uvicorn entrypoint
  `docker/api-entrypoint.sh`
- [ ] T010 Initialize Alembic in `database/migrations/` (`alembic.ini`,
  `env.py` reading `DATABASE_URL`, `script.py.mako`) with one empty baseline revision
- [ ] T011 [P] Create the Next.js shell `apps/web/app/layout.tsx`,
  `apps/web/app/page.tsx`, `apps/web/app/(auth)/signin/page.tsx` placeholder and
  `apps/web/lib/api.ts` (credentials: 'include', CSRF header wiring)
- [ ] T012 Boot the empty stack and record evidence: `docker compose up -d` reaches
  healthy on every service and `make check` exits 0; paste both outputs into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: the Compose stack, both application shells and the `make check` gate
exist.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: schema, configuration, error/log/health substrate, security primitives and
the audit writer — every user story depends on these.

**⚠️ CRITICAL**: no user story work begins until this phase is complete.

- [ ] T013 Implement typed settings in `apps/api/app/core/config.py` (pydantic-settings,
  environment only) — required `ENCRYPTION_KEY` in production, generated dev value,
  no config files read at runtime
- [ ] T014 Create the persistence substrate in `apps/api/app/repositories/base.py` —
  SQLAlchemy `Base`, session factory with `workspace_id` scoping helper, UUID v4
  primary keys, `created_at`/`updated_at`/`deleted_at` conventions (data-model
  Conventions)
- [ ] T015 Implement the error envelope in `apps/api/app/core/errors.py` —
  `{"error":{"code","message","details","request_id"}}`, domain errors → 4xx, unexpected
  → 500 with no internals, one exception handler per class registered in `main.py`
- [ ] T016 Implement structured logging in `apps/api/app/core/logging.py` — JSON lines
  with `ts, level, request_id, route, workspace_id, actor_id, duration_ms`; a
  `request_id` middleware; never log tokens, secrets or private content bodies (P9)
- [ ] T017 Implement security primitives in `apps/api/app/core/security.py` — Argon2id
  hash/verify (password ≥ 12 characters), session cookie issue/revoke/validate
  (`HttpOnly`, `SameSite=Lax`, rotation on privilege change), CSRF token issue/verify
  for cookie-authenticated mutations
- [ ] T018 Define domain enums and the lifecycle constant set in
  `apps/api/app/domain/enums.py` — roles `owner|editor|viewer`, post statuses
  `scheduled|publishing|published|failed|cancelled`, the 11-state content lifecycle, job
  kinds; imported by every layer, importing nothing from FastAPI or SQLAlchemy
- [ ] T019 Implement the append-only audit writer in `apps/api/app/services/audit.py` —
  `record(actor, action, entity_type, entity_id, before, after, request_id)` writing one
  row per mutation; no update or delete path exists (P10, FR-015); reads never write
- [ ] T020 Implement authorization dependencies in `apps/api/app/core/authz.py` —
  `require_role("owner"|"editor"|"viewer")` plus `require_capability(...)`, evaluated in
  the service layer, never only in the UI (P7, FR-003)
- [ ] T021 Write migration `database/migrations/versions/0001_identity.py` creating
  `smos_users`, `smos_workspaces`, `smos_memberships`, `smos_invitations`,
  `smos_auth_attempts` with the constraints from data-model.md (citext unique email,
  unique `(workspace_id, user_id)`, unique `(bucket, window_start)`)
- [ ] T022 Write migration `database/migrations/versions/0002_social_registry.py`
  creating `smos_social_accounts` (unique `(workspace_id, platform, external_id)`,
  `capabilities` jsonb) and `smos_platform_tokens` (AES-GCM columns)
- [ ] T023 Write migration `database/migrations/versions/0003_content.py` creating
  `smos_campaigns`, `smos_content_items` (lifecycle CHECK + `version int`),
  `smos_content_variants` (unique `(content_item_id, platform)`),
  `smos_content_variant_media`, `smos_media_assets` (unique
  `(workspace_id, sha256)`)
- [ ] T024 Write migration `database/migrations/versions/0004_publishing.py` creating
  `smos_platform_posts` (unique `idempotency_key`, status CHECK, `dry_run` default true),
  `smos_jobs` (unique `idempotency_key`), `smos_approval_requests`,
  `smos_notifications` (unique `(user_id, kind, entity_id)`), `smos_audit_events`,
  `smos_settings`
- [ ] T025 Write migration `database/migrations/versions/0005_reserved.py` creating the
  empty-for-Phase-1 tables `smos_comments`, `smos_engagements`,
  `smos_analytics_snapshots`, `smos_agent_runs`, `smos_agent_tasks`,
  `smos_agent_memories`, `smos_automation_rules` so later phases add columns, not tables
- [ ] T026 Create platform capability and limit data in
  `database/seed/platform_definitions.py` — per platform: `publish_text`,
  `publish_image`, `publish_video`, `read_comments`, `reply_comments`, `read_analytics`
  and `platform_limit` characters; seeded by `make seed`, read as data not code
  (FR-004, research R6)
- [ ] T027 Mount the versioned router `apps/api/app/api/v1/router.py` and implement
  `apps/api/app/api/v1/health.py` — `GET /healthz` (process alive) and `GET /readyz`
  (db + redis + minio reachable), returning `200` or `503` naming the failing
  dependency
- [ ] T028 Prove the schema path: run `make db.reset` from a clean database and record
  that all five revisions apply and the seed loads, pasted into
  `specs/001-phase-1-command-center/checklists/requirements.md`
- [ ] T029 [P] Build the test harness `apps/api/tests/conftest.py` — ASGI transport
  client, per-test transaction rollback, factory fixtures (`make_user`, `make_workspace`,
  `make_account`, `make_content`, `make_variant`) and an injectable fake clock
- [ ] T030 [P] Create the unwired AI interface in `apps/api/app/ai/llm_provider.py` —
  `LLMProvider` protocol with no provider configured (P2, research R8)

**Checkpoint**: foundation ready — user story implementation can begin.

---

## Phase 3: User Story 1 - Sign in to the workspace (Priority: P1) 🎯 MVP

**Goal**: an operator signs in with email/password and reaches the command center;
every private route is refused without a valid session, with brute-force protection.

**Independent Test**: with a registered user, submit credentials and confirm the session
and landing page; with no session, confirm every private route refuses; run nine then
ten failed sign-ins and observe the lockout.

### Tests for User Story 1 (MANDATORY — write first, observe failing)

- [ ] T031 [P] [US1] Contract test for `POST /auth/login` (success sets session, wrong
  password and unknown account return byte-identical `401 INVALID_CREDENTIALS`) in
  `apps/api/tests/api/test_auth_login.py`
- [ ] T032 [P] [US1] Integration test sweeping every private route and API path without a
  session → `401` with no data, plus expired/revoked session handling, in
  `apps/api/tests/api/test_auth_guard.py`
- [ ] T033 [P] [US1] Integration test for the lockout policy — nine failures do not lock,
  the tenth in the rolling 15 minutes locks for 15 minutes, the response is
  indistinguishable from a wrong password, expiry restores sign-in, success resets the
  counter, failures and lockout are audited — in
  `apps/api/tests/integration/test_auth_lockout.py`
- [ ] T034 [P] [US1] Integration test for independent per-IP throttling and atomic
  counters under 15 concurrent failed logins (no bypass, other users behind the same IP
  unaffected) in `apps/api/tests/integration/test_auth_throttle.py`

### Implementation for User Story 1

- [ ] T035 [US1] Create repositories `apps/api/app/repositories/users.py`,
  `apps/api/app/repositories/workspaces.py`, `apps/api/app/repositories/memberships.py`
  (email normalised and lowercased at write, per data-model User)
- [ ] T036 [P] [US1] Create `apps/api/app/repositories/auth_attempts.py` with the atomic
  upsert on `(bucket, window_start)` and `locked_until` handling — unique constraint
  `(bucket, window_start)`, `locked_until` set when `failures >=
  policy.lockout_threshold` inside the rolling window (FR-001, R4)
- [ ] T037 [US1] Implement `apps/api/app/services/policy.py` — resolve effective policy
  from constitution defaults overlaid with `workspaces.policy`
  (`approval_required=true`, `spacing_minutes=60`, `lockout_threshold=10`,
  `lockout_window_minutes=15`, `lockout_duration_minutes=15`,
  `retention_soft_delete_days=30`, `retention_notification_days=90`)
- [ ] T038 [US1] Implement `apps/api/app/services/auth.py` — register (owner + workspace
  with a validated IANA `timezone`), login with rolling-window lockout and per-IP
  throttle, logout/session revoke, password change revoking other sessions, audited
  self-service unlock request; every failure audited (FR-001)
- [ ] T039 [US1] Implement routes `apps/api/app/api/v1/auth.py` —
  `POST /auth/register`, `/auth/login`, `/auth/logout`, `GET /auth/me`,
  `POST /auth/password`, `POST /auth/unlock-request` per contracts/api.md §1
- [ ] T040 [US1] Implement the session dependency
  `apps/api/app/core/session.py` — `current_user`, `require_session`, CSRF verification
  on mutations, `401` envelope on absence or expiry
- [ ] T041 [US1] Build the sign-in UI `apps/web/app/(auth)/signin/page.tsx` — generic
  error copy identical for every failure, lockout message with `retry_after_seconds`
- [ ] T042 [US1] Build the private-route guard `apps/web/middleware.ts` and landing
  redirect `apps/web/app/page.tsx` → command center; expired session returns to sign-in
  without submitting pending work
- [ ] T043 [US1] Run `make check`, confirm the four US1 test files pass, and paste the
  command + result into `specs/001-phase-1-command-center/checklists/requirements.md`
  (evidence for SC-007, SC-011)

**Checkpoint**: User Story 1 is fully functional and testable independently — MVP.

---

## Phase 4: User Story 10 - Schedule and publish (Priority: P1)

**Goal**: approved content is scheduled to one or several accounts as independent posts,
dispatched at the appointed instant through the adapter, with recorded outcome, backoff
retries, at-most-once guarantee and conflict-checked spacing.

**Independent Test**: schedule a post, let it come due, confirm dispatch, recorded
outcome and audit entry; force a failure and confirm backoff, final failure and
notification; race two scheduling requests and observe exactly one winning.

> This story's tests seed content, variants and accounts directly through repository
> factories; the end-to-end operator path also needs US3, US8 and US9.

### Tests for User Story 10 (MANDATORY — write first, observe failing)

- [ ] T044 [P] [US10] Contract test for `POST /platform-posts` — `201` returning one row
  per target account with independent schedules, `422 APPROVAL_REQUIRED` for unapproved
  content, `422 CAPABILITY_UNAVAILABLE` for an unsupported capability — in
  `apps/api/tests/api/test_platform_posts_create.py`
- [ ] T045 [P] [US10] Integration test using the fake clock: a post scheduled at T
  dispatches through the adapter within 15 s of T, stores `published` +
  `external_post_id` + `dry_run=true` + audit event, and a replayed job after restart
  does not publish twice — in `apps/api/tests/integration/test_publish_dryrun.py`
- [ ] T046 [P] [US10] Integration test for retry behaviour — backoff at 5 s / 30 s /
  2 min / 10 min, `max_attempts=4`, then `failed` + operator notification, every
  attempt recorded — in `apps/api/tests/integration/test_publish_retry.py`
- [ ] T047 [P] [US10] Integration test for recovery — a job due during downtime is
  reconciled on worker boot rather than lost or fired twice; a worker crash mid-job
  re-claims it after the lease timeout — in
  `apps/api/tests/integration/test_reconcile.py`
- [ ] T048 [P] [US10] Integration test for spacing — same-account post inside the
  60-minute window → `409` naming the conflicting post, account and time; owner
  override accepted only with a non-empty reason and audited; a different account at the
  same instant accepted; two concurrent requests cannot both win — in
  `apps/api/tests/integration/test_spacing.py`

### Implementation for User Story 10

- [ ] T049 [US10] Implement `apps/api/app/domain/state_machine.py` post transitions
  (`scheduled → publishing → published | failed`, `failed → publishing`,
  `scheduled → cancelled`, nothing leaves `published`) as data, rejecting illegal
  transitions with the allowed list
- [ ] T050 [US10] Create `apps/api/app/repositories/platform_posts.py` — fan-out
  creating N rows in one transaction, independent row updates, optimistic status
  transitions, `idempotency_key = f"{content_item_id}:{social_account_id}:{scheduled_at}"`
- [ ] T051 [US10] Implement `apps/api/app/services/scheduling.py` — create/reschedule/
  cancel/bulk-schedule where every resulting post stays independently editable; the
  spacing check runs inside the same transaction as the write (research R3); owner
  override requires `override_reason` and writes both rows + account + timestamps +
  reason to the audit trail (FR-022, FR-023)
- [ ] T052 [US10] Implement `apps/api/app/services/spacing.py` — conflict computation
  over `scheduled` rows on the same account within `±policy.spacing_minutes`, exact
  same-time collisions included, never across accounts
- [ ] T053 [P] [US10] Define the adapter contract in
  `apps/api/app/adapters/protocol.py` — `SocialPlatformAdapter` Protocol plus
  `PublishPayload` and `PublishResult` dataclasses exactly as contracts/events.md §4
- [ ] T054 [P] [US10] Implement `apps/api/app/adapters/dry_run.py` — `DryRunAdapter`
  validating the capability matrix, logging a redacted payload, returning
  `ok=True, dry_run=True, external_post_id=f"dryrun-{idempotency_key[:8]}"`, and
  raising `CAPABILITY_UNAVAILABLE` before any row is created (FR-013, P14)
- [ ] T055 [US10] Create `apps/api/app/repositories/jobs.py` — enqueue with unique
  `idempotency_key`, claim with `FOR UPDATE SKIP LOCKED`, lease timeout re-claim,
  completion only after the post outcome commits (research R2, at-most-once)
- [ ] T056 [US10] Implement `apps/api/app/services/publishing.py` — dispatch one due
  post, record `published`/`failed` outcome with attempt history and `last_error`, apply
  the documented backoff, emit `post.published` / `post.failed` after the outcome row
  commits (FR-011, FR-012)
- [ ] T057 [US10] Implement `services/worker/main.py` — consumer loop over
  `publish_post` jobs calling the service layer (never route handlers), with graceful
  shutdown and structured logs (P5)
- [ ] T058 [US10] Implement `services/scheduler/main.py` — enqueue posts due within the
  lookahead window and run `reconcile_schedule` on boot so overdue jobs are recovered
- [ ] T059 [US10] Implement routes `apps/api/app/api/v1/platform_posts.py` —
  `GET/POST /platform-posts`, `GET/PATCH/DELETE /platform-posts/{id}`,
  `POST /platform-posts/{id}/retry` (honours `Idempotency-Key`) per contracts/api.md §4
- [ ] T060 [P] [US10] Implement the event publisher
  `apps/api/app/services/events.py` — envelope, catalogue and topic naming from
  contracts/events.md §1–2, published after commit, never containing tokens or content
  bodies
- [ ] T061 [US10] Run `make check` and paste the US10 test output (commands + results)
  into `specs/001-phase-1-command-center/checklists/requirements.md` (evidence for
  SC-003, SC-005, SC-010)

**Checkpoint**: scheduling and publication work end to end with dry-run evidence.

---

## Phase 5: User Story 2 - Invite members and assign roles (Priority: P2)

**Goal**: an owner invites people as `owner`, `editor` or `viewer`; invitations are
accepted exactly once and every privileged action is gated by role.

**Independent Test**: invite a user, accept once, then attempt approval/publish as
`editor` and a write as `viewer` and confirm refusal; attempt to remove the last owner
and confirm `409`.

### Tests for User Story 2 (MANDATORY — write first, observe failing)

- [ ] T062 [P] [US2] Contract + integration tests for invitations (single acceptance,
  second acceptance `409`), the role matrix (editor refused approvals/publish, viewer
  refused writes but reads fine) and the last-owner invariant — in
  `apps/api/tests/api/test_members.py`
- [ ] T063 [P] [US2] Integration test for the full authorization matrix across every
  mutating endpoint and both route and service layers in
  `apps/api/tests/integration/test_authz_matrix.py` (SC-007)

### Implementation for User Story 2

- [ ] T064 [US2] Create repositories `apps/api/app/repositories/members.py` and
  `apps/api/app/repositories/invitations.py` — unique `(workspace_id, user_id)`, one
  acceptance per invitation (`accepted_at`), token hash only (P9)
- [ ] T065 [US2] Implement `apps/api/app/services/members.py` — invite, accept, change
  role, remove; refuse any change leaving the workspace ownerless with `409`; audit every
  mutation (FR-002, FR-003)
- [ ] T066 [US2] Implement routes `apps/api/app/api/v1/members.py` —
  `GET/POST /workspaces/{id}/members`, `PATCH/DELETE /workspaces/{id}/members/{user_id}`,
  `POST /workspaces/{id}/invitations` and the accept endpoint
- [ ] T067 [US2] Build the members UI `apps/web/app/settings/members/page.tsx` —
  pending invitations, role selectors disabled for non-owners
- [ ] T068 [US2] Run `make check`, paste US2 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: roles exist and are enforced; US1 and US2 both still pass.

---

## Phase 6: User Story 3 - Connect a social account and see what it can do (Priority: P2)

**Goal**: register a platform account with its status and capability matrix, and refuse
anything the account cannot do.

**Independent Test**: add an account, inspect its capability display, attempt to target
an unsupported capability and confirm rejection naming it; attempt to schedule against a
`needs_reauth` account and confirm refusal.

### Tests for User Story 3 (MANDATORY — write first, observe failing)

- [ ] T069 [P] [US3] Contract tests for `GET/POST /social-accounts` and `GET /platforms`
  — status values, capability matrix returned, unsupported capability rejected with
  `422 CAPABILITY_UNAVAILABLE` naming it, `needs_reauth` refusing scheduling — in
  `apps/api/tests/api/test_social_accounts.py`

### Implementation for User Story 3

- [ ] T070 [P] [US3] Implement AES-GCM helpers in `apps/api/app/core/crypto.py` —
  `encrypt`/`decrypt` under `ENCRYPTION_KEY`, used for `platform_tokens`; no plaintext
  token ever reaches a log, response or the repository tree (P9, FR-019)
- [ ] T071 [US3] Create `apps/api/app/repositories/social_accounts.py` — unique
  `(workspace_id, platform, external_id)`, soft delete, capability jsonb access
- [ ] T072 [US3] Implement `apps/api/app/services/accounts.py` — register (seeding
  capabilities from `database/seed/platform_definitions.py`), list with matrix, refresh
  (`202` in Phase 1, OAuth is Phase 2), disconnect revoking the stored token; reject any
  content targeting an absent capability (FR-004)
- [ ] T073 [US3] Implement routes `apps/api/app/api/v1/social_accounts.py` —
  `GET/POST /social-accounts`, `GET /social-accounts/{id}`,
  `POST /social-accounts/{id}/refresh`, `DELETE /social-accounts/{id}`, `GET /platforms`
- [ ] T074 [US3] Build the accounts UI `apps/web/app/accounts/page.tsx` — connection
  status badges and the capability matrix rendered from data, `needs_reauth` prompt
- [ ] T075 [US3] Run `make check`, paste US3 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: the registry exists; capability enforcement is demonstrable.

---

## Phase 7: User Story 4 - Write content with per-platform variants (Priority: P2)

**Goal**: one base content item with per-platform variants, each with its own body,
media set and limit validation, duplicable into a new draft.

**Independent Test**: create a base item, add a variant, save an over-limit variant and
confirm rejection stating the limit and current length; duplicate and confirm copies.

### Tests for User Story 4 (MANDATORY — write first, observe failing)

- [ ] T076 [P] [US4] Contract tests for `GET/POST /content`, `GET/PATCH /content/{id}`
  (`If-Match: version` → `409` on mismatch), variant creation, over-limit variant →
  `422` naming `platform_limit` and current length, duplicate producing copies not
  references — in `apps/api/tests/api/test_content_variants.py`

### Implementation for User Story 4

- [ ] T077 [US4] Create `apps/api/app/repositories/content.py` — content items and
  variants, unique `(content_item_id, platform)`, `version` increments on write
- [ ] T078 [US4] Implement `apps/api/app/services/content.py` — create, edit with
  optimistic locking, duplicate (copies variant rows), archive, listing filters
  (`status`, `campaign_id`, `q`, `tag`, `archived`); audit every mutation (FR-005,
  FR-007)
- [ ] T079 [US4] Implement `apps/api/app/services/validation.py` — variant body length
  ≤ `platform_limit` else `422` with both numbers; media references validated against
  the workspace (FR-005)
- [ ] T080 [US4] Implement routes `apps/api/app/api/v1/content.py` — `GET/POST /content`,
  `GET/PATCH/DELETE /content/{id}`, `GET/POST /content/{id}/variants`,
  `PUT /content/variants/{id}`, `POST /content/{id}/transition` per contracts/api.md §3
- [ ] T081 [US4] Build the editor UI `apps/web/app/content/[id]/page.tsx` — base body,
  variant tabs per platform with a live character counter against the limit
- [ ] T082 [US4] Run `make check`, paste US4 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: content can be authored per platform with limits enforced.

---

## Phase 8: User Story 8 - Move content through its lifecycle (Priority: P2)

**Goal**: content advances only through legal states; illegal jumps are refused naming
the allowed transitions, and reviewed text cannot change silently.

**Independent Test**: attempt each legal transition, then attempt an illegal one and
confirm the rejection lists what is allowed; try editing an item in `HUMAN_REVIEW`.

### Tests for User Story 8 (MANDATORY — write first, observe failing)

- [ ] T083 [P] [US8] Unit tests exhaustively covering the transition table (every legal
  edge accepted, every illegal edge `409` with the allowed list) in
  `apps/api/tests/unit/test_state_machine.py`
- [ ] T084 [P] [US8] Contract + integration tests: submit-for-review pass-through
  `DRAFT → AI_REVIEWED → HUMAN_REVIEW` creating an `ApprovalRequest`, edits refused in
  `HUMAN_REVIEW` and later — in `apps/api/tests/api/test_lifecycle.py`

### Implementation for User Story 8

- [ ] T085 [US8] Implement `apps/api/app/domain/lifecycle.py` — the transition table and
  guards as pure domain logic (no framework imports), including which states freeze
  editing (FR-009)
- [ ] T086 [US8] Implement `apps/api/app/services/lifecycle.py` — transition execution,
  the submit-for-review pass-through with a note that no model ran (research R8),
  `APPROVED → SCHEDULED` requiring at least one `platform_posts` row, audit +
  `content.transitioned` event per transition
- [ ] T087 [US8] Wire `POST /content/{id}/transition` into
  `apps/api/app/api/v1/content.py` returning `409` with `allowed_transitions` on
  refusal, and run `make check` pasting US8 evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: nothing can skip review.

---

## Phase 9: User Story 9 - Approve, request changes or reject (Priority: P2)

**Goal**: the human gate — approve, request changes or reject, with separation of
duties, and unapproved content cannot be scheduled.

**Independent Test**: approve as an authorised user, request changes as another, confirm
an unapproved item cannot be scheduled and that the submitter cannot approve their own.

### Tests for User Story 9 (MANDATORY — write first, observe failing)

- [ ] T088 [P] [US9] Contract tests for `GET /approvals` and
  `POST /approvals/{id}/{approve|request-changes|reject}` — actor/time/comment recorded,
  content returns to `DRAFT` on request-changes, separation of duties `403`, scheduling
  unapproved → `422 APPROVAL_REQUIRED` — in `apps/api/tests/api/test_approvals.py`

### Implementation for User Story 9

- [ ] T089 [US9] Create `apps/api/app/repositories/approvals.py` — one open request per
  content item, decided_by/decided_at/note persistence
- [ ] T090 [US9] Implement `apps/api/app/services/approvals.py` — approve (item →
  `APPROVED`, emit `content.approved`), request changes (→ `DRAFT` with reviewer note),
  reject (→ `DRAFT`, reason required); submitter cannot decide their own request; policy
  flag `approval_required` enforced at scheduling time (FR-010, P8)
- [ ] T091 [US9] Implement routes `apps/api/app/api/v1/approvals.py` per contracts/api.md
  §5, owner-only for decisions
- [ ] T092 [US9] Build the approval queue UI `apps/web/app/approvals/page.tsx` with
  approve / request-changes / reject actions and the aging pending list
- [ ] T093 [US9] Run `make check`, paste US9 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md` (SC-005)

**Checkpoint**: the approval gate is enforced end to end; US9 + US10 together prove
"no unapproved item publishes".

---

## Phase 10: User Story 13 - Audit who did what (Priority: P2)

**Goal**: every state change traces to actor, time and before/after values, and the
trail is queryable; reads write nothing.

**Independent Test**: perform each mutating action, then query the trail filtered by
entity and date and confirm every action appears with before/after; confirm a read
produced no row.

### Tests for User Story 13 (MANDATORY — write first, observe failing)

- [ ] T094 [P] [US13] Contract tests for `GET /audit-events` — every mutation across the
  suite produced exactly one row with actor/action/entity/before/after/timestamp,
  filters `entity_type`, `entity_id`, `actor_id`, `from`, `to` work, reads write no
  rows, non-owners get `403` — in `apps/api/tests/api/test_audit.py`

### Implementation for User Story 13

- [ ] T095 [US13] Implement the query layer in `apps/api/app/services/audit.py`
  (append to the writer from T019) — filtered, paginated, ordered by `created_at`
  descending, workspace-scoped (FR-015)
- [ ] T096 [US13] Implement route `apps/api/app/api/v1/audit.py` — `GET /audit-events`,
  owner-only, cursor/offset pagination per contracts/api.md §5
- [ ] T097 [US13] Build the audit UI `apps/web/app/audit/page.tsx` — entity/date/actor
  filters and before/after expansion
- [ ] T098 [US13] Run `make check`, paste US13 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md` (SC-004)

**Checkpoint**: the auditability guarantee (P10) is demonstrable.

---

## Phase 11: User Story 5 - Upload media (Priority: P3)

**Goal**: images and video upload with type/size validation, metadata recorded, bytes
only in object storage, and safe retrieval.

**Independent Test**: upload a valid file and retrieve it; upload an oversized or
unsupported file and confirm nothing is stored; confirm a referenced asset refuses
deletion.

### Tests for User Story 5 (MANDATORY — write first, observe failing)

- [ ] T099 [P] [US5] Contract tests for `POST /media`, `GET /media`,
  `GET /media/{id}/url`, `DELETE /media/{id}` — metadata recorded (type, size,
  checksum), `413` over 50 MB and `422` on bad MIME with no stored artifact or partial
  row, delete `409` while referenced by scheduled content — in
  `apps/api/tests/api/test_media.py`

### Implementation for User Story 5

- [ ] T100 [P] [US5] Implement the object-storage adapter
  `apps/api/app/adapters/storage.py` — MinIO/S3 put, get, presigned GET, delete; short
  credential lifetime, no public buckets (FR-006, P9)
- [ ] T101 [US5] Create `apps/api/app/repositories/media.py` — metadata rows, unique
  `(workspace_id, sha256)` reuse, soft delete
- [ ] T102 [US5] Implement `apps/api/app/services/media.py` — MIME allow-list, ≤ 50 MB,
  streamed hash, reject-without-storing on any violation, attach/detach from variants
  via `content_variant_media` (FR-006)
- [ ] T103 [US5] Implement routes `apps/api/app/api/v1/media.py` per contracts/api.md
  §2.4, multipart upload with `413`/`422` handling
- [ ] T104 [US5] Build the media library UI `apps/web/app/media/page.tsx` — upload drop
  zone, library listing, delete with the reference-conflict message
- [ ] T105 [US5] Run `make check`, paste US5 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: media works without touching the database with bytes (P9).

---

## Phase 12: User Story 6 - Group content into campaigns (Priority: P3)

**Goal**: create, edit, duplicate and archive campaigns, attach content, and keep
archived items out of default views.

**Independent Test**: create a campaign, attach content, archive it and confirm default
views hide it while an `archived` filter still reveals it.

### Tests for User Story 6 (MANDATORY — write first, observe failing)

- [ ] T106 [P] [US6] Contract tests for `GET/POST /campaigns`,
  `PATCH/DELETE /campaigns/{id}` — attaching content reported in listings and detail
  views, archived excluded by default and included with `?archived=true` — in
  `apps/api/tests/api/test_campaigns.py`

### Implementation for User Story 6

- [ ] T107 [US6] Create `apps/api/app/repositories/campaigns.py` — `active`/`archived`
  status, soft delete, workspace scoping
- [ ] T108 [US6] Implement `apps/api/app/services/campaigns.py` — create, edit,
  duplicate, archive as the default delete, associate content, default-list exclusion
  (FR-007)
- [ ] T109 [US6] Implement routes `apps/api/app/api/v1/campaigns.py` per
  contracts/api.md §2.5
- [ ] T110 [US6] Build the campaigns UI `apps/web/app/campaigns/page.tsx` — list with
  archived toggle and content counts
- [ ] T111 [US6] Run `make check`, paste US6 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: organisation works without affecting single-item flows.

---

## Phase 13: User Story 7 - See scheduled work on a calendar (Priority: P3)

**Goal**: month and week grids of scheduled content in the workspace timezone, with
rescheduling through a validated form that surfaces conflicts before confirming.

**Independent Test**: schedule posts, view the grid, change one post's time and confirm
the change, its audit entry and the conflict warning for a clashing time.

### Tests for User Story 7 (MANDATORY — write first, observe failing)

- [ ] T112 [P] [US7] Contract tests for `GET /calendar?from&to&account_id` — posts
  grouped on their workspace-local date across a midnight boundary, month/week ranges,
  invalid or missing workspace timezone rejected, timezone change re-displays without
  rescheduling — in `apps/api/tests/api/test_calendar.py`
- [ ] T113 [P] [US7] Integration test: reschedule stores the new UTC instant, writes an
  audit row, and returns `409 SCHEDULE_CONFLICT` before confirmation when another post
  on the same account is inside the window — in
  `apps/api/tests/integration/test_reschedule.py`

### Implementation for User Story 7

- [ ] T114 [US7] Implement `apps/api/app/services/calendar.py` — group `platform_posts`
  by workspace-local calendar date using the IANA identifier (`zoneinfo`), format
  timezone-aware API timestamps, never reinterpret stored instants (FR-026)
- [ ] T115 [US7] Implement route `apps/api/app/api/v1/calendar.py` —
  `GET /calendar` per contracts/api.md §2.5, and the `PUT /workspaces/{id}/timezone`
  settings path rejecting invalid identifiers
- [ ] T116 [US7] Build the calendar UI `apps/web/app/calendar/page.tsx` — month and week
  grids, platform/status chips, reschedule dialog showing the conflict warning before
  confirmation (FR-008)
- [ ] T117 [US7] Run `make check`, paste US7 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: calendar and list views agree on every instant.

---

## Phase 14: User Story 11 - Get notified (Priority: P3)

**Goal**: in-app and email notifications for approval requests and publish outcomes,
sent once — never once per retry — and markable read.

**Independent Test**: trigger an approval request and a publish failure, confirm both
arrive in-app and by email exactly once, then mark them read and confirm the unread
count drops while the record remains retrievable.

### Tests for User Story 11 (MANDATORY — write first, observe failing)

- [ ] T118 [P] [US11] Contract + integration tests for `GET /notifications`,
  `POST /notifications/{id}/read` — approval and final-failure notifications delivered
  in-app and by email, deduplicated across retries via unique
  `(user_id, kind, entity_id)`, read state behaviour — in
  `apps/api/tests/api/test_notifications.py`, asserting against the Mailpit mailbox
- [ ] T119 [P] [US11] Integration test proving a retried publish produces no second
  notification in `apps/api/tests/integration/test_notification_dedupe.py`

### Implementation for User Story 11

- [ ] T120 [US11] Create `apps/api/app/repositories/notifications.py` — rows with
  `read_at`/`emailed_at`, unique `(user_id, kind, entity_id)`
- [ ] T121 [US11] Implement `apps/api/app/services/notifications.py` — create, dedupe,
  mark read, and email dispatch over SMTP (Mailpit in dev), wiring approval-request and
  publish-outcome triggers (FR-014)
- [ ] T122 [US11] Add the `notify` job handler to `services/worker/main.py` — idempotent,
  reusing the outbox from T055
- [ ] T123 [US11] Implement routes `apps/api/app/api/v1/notifications.py` and the
  unread-count indicator `apps/web/components/NotificationsBell.tsx`
- [ ] T124 [US11] Run `make check`, paste US11 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md`

**Checkpoint**: notifications do not require polling.

---

## Phase 15: User Story 12 - Read the command center dashboard (Priority: P3)

**Goal**: totals, today's items, the approval queue and static next-step guidance, all
from live data, with empty states that guide rather than show zeros.

**Independent Test**: with seeded data, load the dashboard and confirm every number
matches a direct query of the underlying records.

### Tests for User Story 12 (MANDATORY — write first, observe failing)

- [ ] T125 [P] [US12] Contract test for `GET /dashboard/summary` — each total equals a
  direct repository count over the same fixture set, p95 < 500 ms on the seeded
  database, and empty-state guidance appears when nothing is scheduled — in
  `apps/api/tests/api/test_dashboard.py`

### Implementation for User Story 12

- [ ] T126 [US12] Implement `apps/api/app/services/dashboard.py` — counts, today's
  posts in the workspace timezone, approval queue, `recommendations` as a static list
  with no model involved (P14, FR-021-adjacent)
- [ ] T127 [US12] Implement route `apps/api/app/api/v1/dashboard.py` —
  `GET /dashboard/summary` per contracts/api.md §5
- [ ] T128 [US12] Build the dashboard UI `apps/web/app/dashboard/page.tsx` — live
  totals, today strip, queue with actionable empty states (US12 scenarios 1–2)
- [ ] T129 [US12] Run `make check`, paste US12 test evidence into
  `specs/001-phase-1-command-center/checklists/requirements.md` (SC-006)

**Checkpoint**: every remaining story is now implemented; cross-cutting work follows.

---

## Phase 16: Polish & Cross-Cutting Concerns

**Purpose**: retention, backup/restore, seed data, contract and security gates, and the
evidence trail (Constitution P9/P11/P12/P13).

- [ ] T130 Implement `apps/api/app/services/retention.py` and its `purge_retention` job
  handler in `services/worker/main.py` — soft-deleted content older than 30 days,
  notifications older than 90 days, `auth_attempts` older than 24 hours, and nothing
  else; abort rather than delete when the store is partially unavailable — plus tests in
  `apps/api/tests/integration/test_retention.py` (FR-024, SC-013)
- [ ] T131 [P] Write `scripts/backup.sh` — coordinated `pg_dump` + MinIO copy + manifest
  of row counts and per-object checksums, encrypted at rest, excluding all plaintext
  credentials; exit non-zero on any integrity failure (FR-025)
- [ ] T132 [P] Write `scripts/restore.sh` — restore database and media into a clean
  Compose project following the documented key-recovery procedure (P9)
- [ ] T133 [P] Write `scripts/verify_restore.sh` — assert row counts, checksums and
  media references; exit 0 only when content, schedules, audit records and media are
  usable; then execute backup → wipe → restore → verify and paste the transcript into
  `specs/001-phase-1-command-center/checklists/requirements.md` (SC-012)
- [ ] T134 [P] Write `database/seed/demo.py` producing the evaluated demo workspace
  (owner account, three social accounts across platforms, campaigns, content in several
  states, scheduled posts, pending approvals) so a fresh install needs no manual data
  entry; wire it to `make seed` (FR-020, SC-001)
- [ ] T135 Regenerate `apps/api/openapi.json` from the router and add
  `scripts/check_contract.sh` comparing it with `docs/API-CONTRACTS.md` expectations,
  failing CI on drift (P11)
- [ ] T136 [P] Run the security sweep: `scripts/scan_secrets.sh` over the tree, cookie
  flag assertions, foreign-`Origin`/CSRF-less mutation `403` tests, two-account
  isolation test, and ciphertext-at-rest query for `platform_tokens` — record outputs
  (P9, AC-5.1–5.9)
- [ ] T137 [P] Write `scripts/bench.sh` and `database/seed/large.py` to measure p50/p95
  for list endpoints (< 300 ms) and the dashboard (< 500 ms, < 1 s cold) on a seeded
  database; paste the numbers into `checklists/requirements.md` (NFR-003/004, SC-006)
- [ ] T138 Execute every scenario in `quickstart.md` §1–§7 end to end and paste each
  command with its result into `specs/001-phase-1-command-center/checklists/requirements.md`
  (P12)
- [ ] T139 [P] Update `README.md` with the four-command onboarding from ARCHITECTURE §8
  and confirm `docs/ACCEPTANCE-CRITERIA.md` rows all cite a recorded run (P12)
- [ ] T140 Run `make check` one final time and record the full output (lint + typecheck +
  tests) in `checklists/requirements.md`; confirm no test is skipped, weakened or
  suppressed
- [ ] T141 Verify P1 end to end: `docker compose down -v && docker compose up -d &&
  make db.reset && make check` from a clean clone, recording the transcript in
  `specs/001-phase-1-command-center/checklists/requirements.md`
- [ ] T142 Confirm no secret, token, credential or PII landed in the tree
  (`git ls-files | xargs grep`-style scan plus `./.env` ignore check) and record the
  result in `specs/001-phase-1-command-center/checklists/requirements.md` (P9)
- [ ] T143 Run `/speckit-converge` against the spec: verify each FR and SC, append any
  remaining work to this file rather than declaring done early, and mark the checklist
  items with their evidence (P13)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies — starts immediately.
- **Foundational (Phase 2)**: depends on Setup — **blocks every user story**.
- **User stories (Phase 3+)**: all depend on Foundational.
  - Delivery order follows spec priority: **US1 (P1) → US10 (P1) → US2, US3, US4, US8,
    US9, US13 (P2) → US5, US6, US7, US11, US12 (P3)**.
  - Within P2, US8 → US9 is the only ordered pair (approval needs the lifecycle);
    US9 → US10 is an integration edge, not a build-order edge.
- **Polish (Phase 16)**: depends on the stories it evidences; T131–T134, T136, T139 can
  start as soon as the code they test exists.

### User Story Dependencies

| Story | Can start after | Notes |
|---|---|---|
| US1 (P1) | Foundational | no story dependencies — the gate for everything |
| US10 (P1) | Foundational | tests seed content/accounts through factories; full operator path integrates US3/US8/US9 |
| US2 (P2) | Foundational | uses US1's session dependency |
| US3 (P2) | Foundational | independent |
| US4 (P2) | Foundational | independent |
| US8 (P2) | Foundational | operates on US4's content rows |
| US9 (P2) | US8 (needs the approval request to exist) | independently testable once US8 lands |
| US13 (P2) | Foundational | audits every story's mutations |
| US5 (P3) | Foundational | independent |
| US6 (P3) | Foundational | independent |
| US7 (P3) | US10 (reads `platform_posts`) | independently testable with seeded posts |
| US11 (P3) | US9 + US10 (its triggers) | tests can inject events directly |
| US12 (P3) | Foundational | summarises other stories; needs seed data |

### Within Each User Story

1. Tests are written first and observed **failing**.
2. Repositories/models → services → routes → UI.
3. Story's `make check` + evidence task before moving on.

### Parallel Opportunities

- All `[P]` tasks within a phase touch different files and can run concurrently.
- Once Foundational completes: US2/US3/US4/US5/US6 can proceed in parallel; US13
  (audit query) and US12 (dashboard) can run against any completed story's mutations.
- Migration tasks T021–T025 are sequential by design (revision chain) but can be
  authored in one pass.

---

## Parallel Example: User Story 1

```bash
# Launch together (no shared files):
Task: "Contract test for POST /auth/login in apps/api/tests/api/test_auth_login.py"
Task: "Integration test for private-route refusal in apps/api/tests/api/test_auth_guard.py"
Task: "Integration test for lockout policy in apps/api/tests/integration/test_auth_lockout.py"
Task: "Integration test for IP throttle concurrency in apps/api/tests/integration/test_auth_throttle.py"

# Then the implementation tasks that share ordering:
Task: "repositories/auth_attempts.py with atomic upsert"
Task: "services/policy.py effective policy resolution"
```

---

## Implementation Strategy

### MVP first (US1 only)

1. Complete Phase 1: Setup.
2. Complete Phase 2: Foundational (**critical — blocks all stories**).
3. Complete Phase 3: User Story 1.
4. **STOP and validate**: run the four US1 test files and the quickstart §0–§1 steps.
5. Demo: sign in, land on the command center.

### Incremental delivery

1. Setup + Foundational → foundation ready.
2. US1 → sign-in works (MVP checkpoint).
3. US10 → dry-run publication works (the product's core outcome).
4. US2/US3/US4/US8/US9/US13 → the governed authoring pipeline.
5. US5/US6/US7/US11/US12 → the operational surface.
6. Polish → retention, backup, contract and evidence gates.

Each story is demoed with `make check` green before the next begins.

---

## Notes

- `[P]` = different files, no dependency on incomplete tasks.
- `[Story]` labels map tasks to spec.md user stories for traceability.
- Commit after each task or logical group; stop at any checkpoint to validate the story
  independently.
- Every test task is MANDATORY (Constitution P12) — none may be dropped to speed up
  delivery.
- Avoid vague tasks, same-file conflicts, and cross-story dependencies that break
  independence.
