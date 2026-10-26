# Contracts: API — Social Command Center (Phase 1)

**Feature**: `001-phase-1-command-center` · **Date**: 2026-10-26 · **Phase 1 output**

This is the endpoint contract for this feature, written contract-first (P11) and folded
into [docs/API-CONTRACTS.md](../../docs/API-CONTRACTS.md), which was reconciled to the
`platform_posts` model in the same step. Conventions (base path `/api/v1`, cookie
session + CSRF header, error envelope, UUIDs, ISO-8601 UTC, `Idempotency-Key`,
pagination) are inherited from that document and not repeated here.

## 1. Identity & workspace

| Method | Path | Role | Notes |
|---|---|---|---|
| POST | `/auth/register` | public | creates owner + workspace; `timezone` required |
| POST | `/auth/login` | public | `429` with `ACCOUNT_LOCKED` after 10 failures / 15 min; per-IP throttled independently |
| POST | `/auth/logout` | authenticated | revokes session |
| GET | `/auth/me` | authenticated | user, workspace, role, capabilities |
| POST | `/auth/password` | authenticated | change password, revokes other sessions |
| POST | `/auth/unlock-request` | public | audited self-service recovery for a locked account |
| GET/POST | `/workspaces/{id}/members` | owner | list, invite |
| PATCH/DELETE | `/workspaces/{id}/members/{user_id}` | owner | role change, removal; `409` if it would remove the last owner |

```jsonc
// POST /api/v1/auth/login → 401, identical for wrong password and unknown account
{ "error": { "code": "INVALID_CREDENTIALS",
  "message": "Email or password is incorrect.", "details": {}, "request_id": "…" } }

// POST /api/v1/auth/login → 429 after the tenth failure in the rolling window
{ "error": { "code": "ACCOUNT_LOCKED",
  "message": "Too many failed attempts. Try again later.",
  "details": { "retry_after_seconds": 512 }, "request_id": "…" } }
```

## 2. Social accounts

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/social-accounts` | viewer+ | includes the `capabilities` matrix |
| POST | `/social-accounts` | owner | `{platform, account_name, external_id}` → `connected`; capabilities seeded from the platform definition |
| GET | `/social-accounts/{id}` | viewer+ | status, capabilities, `last_sync_at` |
| POST | `/social-accounts/{id}/refresh` | owner | Phase 2 performs the OAuth exchange; Phase 1 returns `202` and sets `needs_reauth` handling in place |
| DELETE | `/social-accounts/{id}` | owner | soft delete, revokes stored token |
| GET | `/platforms` | viewer+ | supported platforms and their capability matrices |

## 3. Content, variants, media, campaigns, calendar

Unchanged from [docs/API-CONTRACTS.md](../../docs/API-CONTRACTS.md) §2.3–2.5 except:
optimistic locking on `PATCH /content/{id}` uses `If-Match: <version>`; and the
scheduling endpoints move to `/platform-posts` (below).

## 4. Platform posts *(replaces `/scheduled-posts` and `/published-posts`)*

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/platform-posts` | viewer+ | filters: `status`, `content_item_id`, `account_id`, `from`, `to`, `dry_run` |
| POST | `/platform-posts` | editor+ | fan-out create (below); `201` with one row per target account |
| GET | `/platform-posts/{id}` | viewer+ | single post incl. attempts and outcome |
| PATCH | `/platform-posts/{id}` | editor+ | reschedule (`scheduled_at`); re-runs the spacing check; audited |
| DELETE | `/platform-posts/{id}` | editor+ | cancel a `scheduled` post → `cancelled`, audited |
| POST | `/platform-posts/{id}/retry` | owner | manual re-dispatch of a `failed` post |
| GET | `/calendar` | viewer+ | `?from=&to=&account_id=` grouped by workspace-local date |

```jsonc
// POST /api/v1/platform-posts → 201  (fan-out, FR-022)
{ "content_item_id": "5f0e…",
  "social_account_ids": ["acc-li…", "acc-x…", "acc-ig…"],
  "schedule": { "mode": "same", "scheduled_at": "2026-10-30T09:00:00+01:00" },
  "override_reason": null }

// schedule.mode may be "same" or "per_account":
{ "schedule": { "mode": "per_account",
  "items": [ { "social_account_id": "acc-li…", "scheduled_at": "2026-10-30T09:00:00+01:00" },
             { "social_account_id": "acc-x…",  "scheduled_at": "2026-10-30T10:30:00+01:00" } ] } }

// 201 — one independent row per target; per-row results, never all-or-nothing
{ "items": [
    { "id": "pp-1…", "social_account_id": "acc-li…", "status": "scheduled",
      "scheduled_at": "2026-10-30T08:00:00Z", "dry_run": true },
    { "id": "pp-2…", "social_account_id": "acc-x…", "status": "scheduled",
      "scheduled_at": "2026-10-30T09:30:00Z", "dry_run": true },
    { "id": "pp-3…", "social_account_id": "acc-ig…", "status": "scheduled",
      "scheduled_at": "2026-10-30T17:00:00Z", "dry_run": true } ],
  "rejected": [] }

// 422 — approval required (policy default, P8)
{ "error": { "code": "APPROVAL_REQUIRED",
  "message": "Content must be APPROVED before scheduling.",
  "details": { "current_status": "HUMAN_REVIEW" }, "request_id": "…" } }

// 409 — spacing conflict (FR-023); owner may retry with override_reason
{ "error": { "code": "SCHEDULE_CONFLICT",
  "message": "Another post targets this account within the 60-minute spacing window.",
  "details": { "conflicting_post_id": "pp-9…",
               "conflicting_scheduled_at": "2026-10-30T09:30:00Z",
               "spacing_minutes": 60 }, "request_id": "…" } }

// 422 — capability unavailable on the target account (FR-004, P14)
{ "error": { "code": "CAPABILITY_UNAVAILABLE",
  "message": "This account cannot publish video.",
  "details": { "capability": "publish_video", "social_account_id": "acc-ig…" },
  "request_id": "…" } }
```

Retried work: `POST /platform-posts/{id}/retry` honours `Idempotency-Key`; the outcome
of every attempt (success or failure) is visible on the post and in the audit trail.
`dry_run` is `true` on every Phase 1 record and on the corresponding domain events.

## 5. Approvals, notifications, audit, settings, dashboard

Unchanged from the canonical contract, with these clarifications for this feature:

- `POST /approvals/{id}/approve` → `200` moves the item to `APPROVED`, writes one audit
  event, and emits `content.approved`.
- `GET /audit-events` supports `?entity_type=&entity_id=&actor_id=&from=&to=`, ordered
  by `created_at` descending, and is owner-only.
- `GET/PUT /settings/policy` exposes exactly the policy keys listed in the data model
  (`approval_required`, `spacing_minutes`, lockout thresholds, retention windows) with
  constitution defaults shown alongside overrides.
- `GET /dashboard/summary` returns `{totals, today, approval_queue, recommendations}`
  where `recommendations` is a static list in Phase 1 (no AI, P14).

## 6. Operational contract (scripts, not HTTP)

FR-025 is delivered as repository scripts, not endpoints, because they run outside the
application trust boundary:

```bash
scripts/backup.sh  --out <dir>     # pg_dump + MinIO copy + manifest; exit 1 on failed integrity check
scripts/restore.sh --from <dir>    # restores into a clean Compose project
scripts/verify_restore.sh          # asserts counts, checksums and media references; exit 0 = usable
```

The manifest records row counts per table and a checksum per media object; a mismatch
in either fails the backup. Secrets and encryption keys are excluded from the backup by
design; recovery of `ENCRYPTION_KEY` is a documented operator procedure (P9).

## 7. Health

`GET /healthz` (process alive) and `GET /readyz` (db, redis, minio reachable) are
public and unauthenticated; both return `200` or `503` with the failing dependency
named in `details`.
