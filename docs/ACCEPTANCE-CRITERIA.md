# Social OS — Acceptance Criteria (Phase 1)

**Status:** Draft · **Version:** 0.1 · **Date:** 2026-10-25
Every criterion is provable by a command. Record the command and its result in the
Phase 1 feature `checklist.md` before declaring Phase 1 done (Constitution P12).

---

## AC-1 Clean installation

| ID | Criterion | Evidence |
|---|---|---|
| AC-1.1 | `git clone` → `cp .env.example .env` → `docker compose up -d` succeeds with no other manual step | terminal transcript |
| AC-1.2 | All Phase 1 services report `healthy` | `docker compose ps` |
| AC-1.3 | `GET :8000/healthz` → `200`, `GET :8000/readyz` → `200` | `curl -s -o /dev/null -w '%{http_code}'` |
| AC-1.4 | `GET :3000` returns HTML for the login screen | `curl` + screenshot |
| AC-1.5 | Build is reproducible: second `docker compose build` produces no errors and CI matches local | CI log |

## AC-2 Persistent data

| ID | Criterion | Evidence |
|---|---|---|
| AC-2.1 | Seeded content survives `docker compose down` + `docker compose up -d` (data volume intact) | before/after row counts |
| AC-2.2 | Media uploaded to MinIO survives container rebuild; blob byte-identical (sha256) | `sha256sum` comparison |
| AC-2.3 | `alembic upgrade head` on an empty database reaches the same schema hash as CI | schema dump diff |
| AC-2.4 | No domain table stores file blobs (metadata only, ARCHITECTURE §4) | schema inspection |

## AC-3 Functional walkthrough (the Phase 1 pipeline)

Run against seeded data; each step asserts on API response *and* visible UI state.

| ID | Criterion | Evidence |
|---|---|---|
| AC-3.1 | Register → login → dashboard renders live numbers | E2E test |
| AC-3.2 | Connect a social account → capability matrix displayed | E2E test |
| AC-3.3 | Create content + LinkedIn variant → variant saved with length validation | API test |
| AC-3.4 | Upload media → `MediaAsset` row + retrievable object | API test |
| AC-3.5 | Attach content to a campaign → campaign view lists it | E2E test |
| AC-3.6 | `DRAFT → HUMAN_REVIEW` creates an approval request; illegal transition returns `409` | API test |
| AC-3.7 | `editor` attempting approval → `403`; `owner` approving → `APPROVED` + audit row | API test |
| AC-3.8 | Scheduling unapproved content → `422 APPROVAL_REQUIRED`; approved → `201` | API test |
| AC-3.9 | Calendar shows the scheduled post on the correct date | E2E test |
| AC-3.10 | Publish worker dispatches at `scheduled_at` (within 15 s) via `DryRunAdapter`, writes one `platform_posts` row per target account with `dry_run=true` and an audit event | integration test with fake clock |
| AC-3.11 | Simulated adapter failure → retries at documented backoff → `FAILED` + operator notification | integration test |
| AC-3.12 | Duplicate `Idempotency-Key` does not double-publish | API test |
| AC-3.13 | Notification appears in-app and in Mailpit | E2E test + mail log |
| AC-3.14 | Audit trail answers "who approved and what was sent" for a published post | query + assertion |

## AC-4 Tests and quality gates

| ID | Criterion | Evidence |
|---|---|---|
| AC-4.1 | `make check` (lint + typecheck + tests) exits 0 locally and in CI | CI run |
| AC-4.2 | Unit tests cover the state machine's legal and illegal transitions exhaustively | coverage report |
| AC-4.3 | Every `US-xxx` acceptance criterion in PHASE-1-SPEC §4 maps to at least one test | traceability table in checklist |
| AC-4.4 | API contract and running server agree (`openapi.json` diff clean) | contract test |
| AC-4.5 | No test is skipped or weakened to pass; failures are fixed, not suppressed | review |

## AC-5 Secure configuration

| ID | Criterion | Evidence |
|---|---|---|
| AC-5.1 | Unauthenticated request to every private route → `401` (route sweep test) | automated sweep |
| AC-5.2 | Cross-workspace access → `404`/`403`, never data (two-account isolation test) | automated test |
| AC-5.3 | Foreign `Origin`/CSRF-less mutation → `403` | automated test |
| AC-5.4 | `grep`-style secret scan of the repo finds no keys/tokens; CI enforces it | scanner output |
| AC-5.5 | DB dump shows ciphertext for `platform_tokens`, never plaintext | query |
| AC-5.6 | `.env` is git-ignored; `.env.example` contains placeholders only | `git status`/`git check-ignore` |
| AC-5.7 | Errors returned to clients contain no stack traces or internals | negative test |
| AC-5.8 | Failed-sign-in lockout: 9 failures do not lock, the 10th in a rolling 15 min locks for 15 min (`429`), errors are identical for unknown account and wrong password, and lockout/unlock are audited | API test |
| AC-5.9 | Per-IP throttling is independent of the account lockout, and concurrent failed logins cannot bypass either threshold | API test + concurrency test |

## AC-6 Scheduling reliability

| ID | Criterion | Evidence |
|---|---|---|
| AC-6.1 | Job fires within 15 s of `scheduled_at` under normal load (NFR-003/004) | timing test |
| AC-6.2 | Restarting `api` mid-wait still publishes (queue is durable, not in-memory) | chaos test |
| AC-6.3 | `reconcile_schedule` on worker boot enqueues posts missed during downtime | integration test |
| AC-6.4 | Same post never dispatched twice (idempotency across worker restarts) | integration test |
| AC-6.5 | Clock/timezone handling: DST-crossing schedules resolve correctly | unit tests |
| AC-6.6 | Fan-out to N accounts creates N independent `platform_posts`, each with its own schedule, retry and outcome | API test |
| AC-6.7 | Same-account spacing conflict → `409` naming the clash; owner override requires a reason and is audited; a different account at the same time is accepted | API test |
| AC-6.8 | Calendar and API present times in the workspace IANA timezone; changing it re-displays without rescheduling | E2E + API test |

## AC-7 Failure recovery

| ID | Criterion | Evidence |
|---|---|---|
| AC-7.1 | `docker compose stop db` → API returns `503` on `readyz`, recovers after restart without data loss | manual/scripted run |
| AC-7.2 | Worker crash mid-job → job is re-claimed and completes exactly once | integration test |
| AC-7.3 | Adapter failure → bounded retries, then `FAILED` + notification, never silent loss | integration test |
| AC-7.4 | Corrupt/oversized upload rejected without partial row or orphan blob | API test |
| AC-7.5 | `scripts/backup.sh` succeeds with a verified manifest; `scripts/restore.sh` into a clean environment reproduces content, schedules, audit rows and media (`verify_restore.sh` exit 0) | scripted run |
| AC-7.6 | Retention purge removes only soft-deleted content > 30 d, notifications > 90 d and `auth_attempts` > 24 h — nothing else | integration test |
| AC-7.5 | Migration rollback (`downgrade -1` → `upgrade head`) succeeds on seeded data | migration test |

## AC-8 Reproducible deployment

| ID | Criterion | Evidence |
|---|---|---|
| AC-8.1 | Images pinned (no `latest` tags) and lockfiles committed | `docker compose config` |
| AC-8.2 | CI pipeline runs `make check`, build, and the AC-1 smoke sequence on every PR | CI log |
| AC-8.3 | `make check` is the same command locally and in CI | Makefile + CI config |
| AC-8.4 | Clean-clone walkthrough performed by someone other than the author within 15 minutes | sign-off note |

## Sign-off

Phase 1 is accepted when: every AC above is `pass` with recorded evidence, the Phase 1
`checklist.md` is complete, and the constitution's quality bar (§C) holds. Anything not
provable is recorded as a limitation, never as a pass.
