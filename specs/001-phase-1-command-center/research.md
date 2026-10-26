# Research: Social Command Center (Phase 1)

**Feature**: `001-phase-1-command-center` · **Date**: 2026-10-26 · **Phase 0 output**

Stack-level unknowns were already pinned by [docs/ARCHITECTURE.md](../../docs/ARCHITECTURE.md);
this file resolves the eight decisions the clarified spec introduced. Every
`NEEDS CLARIFICATION` from Technical Context is resolved here — none remain.

Format: **Decision** → **Rationale** → **Alternatives considered**.

---

## R1 — Fan-out scheduling model

**Decision**: one `platform_posts` row per target social account, created atomically in
a single transaction when the operator multi-selects accounts. The parent
`ContentItem` stays the source of truth; each row owns its own variant, schedule,
approval reference, retry policy, attempt history, status, external reference and
outcome. Bulk actions (shared schedule, bulk approval) write N independent rows and
report per-row results.

**Rationale**: matches the clarified spec (FR-022, US-010 scenarios 6–7) and the
canonical model in the source document (`PlatformPost` is already the adapter payload
type in `docs/API-CONTRACTS.md` §4). Independence is what lets one failed network
retry without touching its siblings, and what Phase 7 analytics attaches to.

**Alternatives considered**:
- *Atomic multi-account publish* — rejected: partial platform failures make "all or
  nothing" unimplementable, and per-platform timing differs.
- *Keep `scheduled_posts` + `published_posts` as two tables* — rejected: the lifecycle
  of one target post would be split across two rows with divergent statuses.
- *A parent post with child rows* — rejected as a separate concept: the
  `ContentItem → platform_posts` relationship already is the parent/child structure.

## R2 — At-most-once publication across restarts

**Decision**: a durable outbox table (`jobs`) in Postgres is the system of record for
scheduled work; Redis carries the wake-up signal only. The worker claims work with
`SELECT … FOR UPDATE SKIP LOCKED`, guards each attempt with an idempotency key of
`(platform_post_id, attempt_group)`, and re-checks the row's status before dispatch so a
post already marked `published` or `failed` is never dispatched twice. On boot the
scheduler re-enqueues every row whose `scheduled_at` has passed and is still
`scheduled`.

**Rationale**: satisfies FR-012 and US-010 scenarios 4–5 (no double publish, overdue
jobs reconciled). Claiming inside the database transaction is what makes the guarantee
hold when the host dies between "decided to publish" and "recorded the result".

**Alternatives considered**:
- *Redis-only queue (BRPOP)* — rejected: a Redis flush or crash loses jobs, failing
  "none are lost".
- *APScheduler persistent job store* — rejected: a second source of truth competing
  with Postgres rows; reconciliation would be ambiguous.
- *Celery/RQ with a results backend* — rejected: heavy dependency for four job types
  when the outbox + worker loop is ~100 lines; P1 prefers the boring path.
- *Postgres `LISTEN/NOTIFY` only* — retained as the wake-up optimisation layered on
  top of the outbox, not as the queue itself.

## R3 — Conflict and spacing enforcement under concurrency

**Decision**: the spacing check runs inside the same transaction as the schedule write,
and that transaction first takes a row lock on the target `social_accounts` row
(`SELECT … FOR UPDATE`). Serialized per account, the service then queries for existing
`scheduled` rows on that account within `±spacing_window` of the proposed instant and
rejects with `409` naming the conflicting post, account and time. The window is
workspace configuration (default 60 minutes). An owner override writes
`override_reason` plus an audit event carrying both posts, the account, timestamps and
the decision. Different accounts never contend.

**Rationale**: satisfies FR-023 and SC-010 — two racing requests for the same account
cannot both pass, because the second waits on the row lock and then sees the first
row. Re-checking at confirmation time (not just at form render) is free once the check
lives in the write transaction.

**Alternatives considered**:
- *Postgres exclusion constraint on `tstzrange`* — rejected for now: the range width
  depends on a configurable window, which a fixed constraint cannot express without
  re-writing the constraint on every config change. Kept as a future hardening option
  if spacing ever becomes a fixed global constant.
- *Check in the API layer before insert* — rejected: check-then-insert races under
  concurrency; this is exactly the bypass SC-010 tests for.
- *Redis lock per account* — rejected: adds a distributed-lock dependency while a
  database lock already serialises the same resource.

## R4 — Brute-force counters that survive a restart

**Decision**: `auth_attempts` in Postgres, updated with an atomic upsert inside the
authentication transaction: per-identity (`email`) and per-IP buckets keyed by a
rolling window. Ten identity failures in 15 minutes locks the identity for 15 minutes;
the IP bucket throttles independently. A successful sign-in resets the identity
bucket. Lockout, throttle, unlock and recovery writes all emit audit events, and every
failure path returns the identical `INVALID_CREDENTIALS` envelope.

**Rationale**: FR-001/SC-011 require that no sequence of concurrent requests bypasses a
threshold and that a cache flush cannot simply unlock an attacker. A database row
updated atomically gives both properties with no new infrastructure.

**Alternatives considered**:
- *Redis counters* — rejected: ephemeral; a restart or `FLUSHDB` clears an attacker's
  history, and multi-worker atomicity still needs care.
- *Rate-limit middleware only (per IP)* — rejected: shared NATs punish innocent users
  and rotating IPs bypass it; the spec requires both dimensions.
- *Lockout until admin reset* — rejected in clarification Q3: recovery must be
  self-service and audited, not a support event.

## R5 — Timezone handling

**Decision**: every stored timestamp is `timestamptz` in UTC. Each workspace holds one
IANA timezone identifier (validated at write, rejected if missing or invalid).
Calendar rendering, scheduling input and all displayed times use that zone; the API
returns ISO-8601 values with offsets; the worker compares UTC instants only and never
reads server-local time. Changing the workspace timezone updates display, never a
stored instant.

**Rationale**: FR-026 and SC-003 — "publishes at the intended instant, not an hour
off" — plus correct daylight-saving behaviour, which a fixed `UTC+1` offset cannot
give.

**Alternatives considered**:
- *Store local wall-clock time* — rejected: ambiguous across DST and unrecoverable.
- *Per-user timezone* — deferred by clarification Q5; would make one shared calendar
  slot mean two things.
- *Client-only conversion* — rejected: server rendering and the worker must agree
  independently of the browser.

## R6 — Capability enforcement without a platform

**Decision**: `social_accounts.capabilities` is stored as a JSON capability matrix
(`publish_text`, `publish_image`, `publish_video`, `read_comments`, `reply_comments`,
`read_analytics`) seeded from the platform definition at registration. The service
validates a variant's target against that matrix before accepting a schedule and
returns `422` naming the unavailable capability. `DryRunAdapter` implements the same
`Protocol` as future real adapters and marks every result `dry_run=true`.

**Rationale**: FR-004/FR-013, P3 and P14 — capability is *data*, so Phase 2 changes
rows, not code, and Phase 1 can prove rejection paths with no third-party API.

**Alternatives considered**:
- *Capability checks in each adapter* — rejected: pushes policy into the edge and makes
  core behaviour untestable offline.
- *Hard-coded per-platform rules in services* — rejected: violates P3 directly.

## R7 — Backup, restore and retention

**Decision**: `scripts/backup.sh` takes a consistent pair of snapshots — `pg_dump` of
the database and a copy of the MinIO data directory — in a defined order, writes a
manifest with row counts and per-object checksums, and exits non-zero if the integrity
check fails. `scripts/restore.sh` restores into a clean Compose project;
`scripts/verify_restore.sh` asserts content, schedules, approvals, audit rows and media
references all resolve. Encryption keys and secrets are never written into the backup;
their recovery is a documented operator procedure. Retention is a worker job that
purges soft-deleted content past 30 days and notifications past 90, in batches, in a
dry-run-capable form, aborting rather than partially deleting when the store is
unavailable.

**Rationale**: FR-024/FR-025 and SC-012/SC-013 — the operator chose "also ship
backup/restore", and a backup that cannot be verified is not a backup.

**Alternatives considered**:
- *Volume snapshots only, untested* — rejected: excluded by the chosen option.
- *WAL archiving / PITR* — deferred: valuable but beyond Phase 1 scope; the manifest
  approach leaves room to add it.
- *Postgres-native TTL* — does not exist; retention must be application-driven.

## R8 — The AI-review step with no AI

**Decision**: the transition `DRAFT → AI_REVIEWED → HUMAN_REVIEW` is executed by the
lifecycle service as a single pass-through in Phase 1: submitting for review writes both
states with a note recording that no model ran, and creates the approval request. The
`LLMProvider` protocol exists in `apps/api/app/ai/` with a `NullProvider` that raises if
called.

**Rationale**: FR-009 keeps the exact state machine of later phases intact (so Phase 3
changes behaviour, not schema) while P2/P6 keep the local-AI path untouched.

**Alternatives considered**:
- *Skip the state entirely in Phase 1* — rejected: the state machine and its audit
  history would change shape later, breaking the audit trail's meaning.
- *Ship a dummy model call* — rejected: pointless latency and a fake capability
  marker (P14).

---

## Resolved unknowns checklist

| Technical Context field | Status |
|---|---|
| Language/Version | Resolved (Python 3.12 / TypeScript 5) |
| Primary Dependencies | Resolved (FastAPI stack, no new dependency added) |
| Storage | Resolved (Postgres + Redis + MinIO, `platform_posts` + `auth_attempts` + `jobs`) |
| Testing | Resolved (pytest with fake clock; Vitest/Playwright for web) |
| Target Platform | Resolved (Compose, amd64/arm64, offline) |
| Performance Goals | Resolved (from NFR-003/004, SC-003/006) |
| Constraints | Resolved (P1–P14 + clarified defaults) |
| Scale/Scope | Resolved (single workstation, 3 roles, 26 FRs) |

No `NEEDS CLARIFICATION` markers remain in Technical Context or this document.
