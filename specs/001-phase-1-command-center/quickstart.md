# Quickstart: validating Social Command Center (Phase 1)

**Feature**: `001-phase-1-command-center` · **Date**: 2026-10-26 · **Phase 1 output**
Run from the repository root. Every scenario ends with an assertion you can read; the
output of these commands is the evidence recorded in
[checklists/requirements.md](checklists/requirements.md) and the Phase 1
`docs/ACCEPTANCE-CRITERIA.md` (P12).

## 0. One-time setup

```bash
cp .env.example .env          # generates ENCRYPTION_KEY locally; never committed (P9)
docker compose up -d          # db, redis, minio, api, worker, scheduler, web, mailpit
make db.reset                 # alembic upgrade head + seed demo workspace
make check                    # lint + typecheck + tests — identical to CI
```

Expect: all containers `healthy`, `make check` exit 0, `GET /readyz` → `200`.
**AC-1 / SC-001** — a clean clone needs no step beyond these four.

## 1. Boot to a signed-in, seeded command center (SC-001)

```bash
curl -sf localhost:8000/healthz && echo            # 200
open http://localhost:3000                          # web UI
# sign in with the seeded owner from database/seed/, then:
curl -sf -b cookies.txt localhost:8000/api/v1/dashboard/summary | jq '.totals'
```

Expect: non-zero seeded counts matching the seed script's printed summary; dashboard
renders in the workspace timezone (`Africa/Lagos` by default).

## 2. End-to-end content flow, under 10 minutes (SC-002, SC-005, SC-004)

```bash
# 1. create content
CID=$(curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/content \
  -d '{"title":"AI in manufacturing","base_body":"Field notes from the plant floor."}' | jq -r .id)
# 2. variant + transition DRAFT → HUMAN_REVIEW
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/content/$CID/variants \
  -d '{"platform":"linkedin","body":"…"}' >/dev/null
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/content/$CID/transition \
  -d '{"to_state":"HUMAN_REVIEW"}' | jq -r .status          # HUMAN_REVIEW
# 3. schedule BEFORE approval → must be refused
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/platform-posts \
  -d '{"content_item_id":"'$CID'","social_account_ids":["'$LI'"],"schedule":{"mode":"same","scheduled_at":"2026-10-30T09:00:00+01:00"}}' \
  | jq -r .error.code                                       # APPROVAL_REQUIRED (422)
# 4. approve (as owner), then schedule a fan-out to 3 accounts
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/approvals/$CID/approve >/dev/null
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/platform-posts \
  -d '{"content_item_id":"'$CID'","social_account_ids":["'$LI'","'$X'","'$IG'"],"schedule":{"mode":"same","scheduled_at":"2026-10-30T09:00:00+01:00"}}' \
  | jq '{n:(.items|length), statuses:[.items[].status]}'      # n:3, all "scheduled"
```

Expect: 3 **independent** `platform_posts` rows, each with its own schedule/outcome;
every step present in `GET /audit-events?entity_id=$CID`. **SC-004, SC-005**

## 3. Spacing conflict, override, and independence per account (SC-010)

```bash
# B targets the same account 30 min after A → 409
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/platform-posts \
  -d '{"content_item_id":"'$CID'","social_account_ids":["'$LI'"],"schedule":{"mode":"same","scheduled_at":"2026-10-30T09:30:00+01:00"}}' \
  | jq '{code:.error.code, conflict:.error.details.conflicting_scheduled_at}'   # SCHEDULE_CONFLICT
# same time on a DIFFERENT account → accepted (per-account rule)
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/platform-posts \
  -d '{"content_item_id":"'$CID'","social_account_ids":["'$X'"],"schedule":{"mode":"same","scheduled_at":"2026-10-30T09:30:00+01:00"}}' \
  | jq -r .items[0].status                                # scheduled
# owner override with a reason → accepted + audited
curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/platform-posts \
  -d '{"content_item_id":"'$CID'","social_account_ids":["'$LI'"],"schedule":{"mode":"same","scheduled_at":"2026-10-30T09:30:00+01:00"},"override_reason":"Campaign launch window"}' \
  | jq -r .items[0].status                                # scheduled
# the same override WITHOUT a reason → 422/409 (never silent)
```

Automated coverage must also fire two concurrent scheduling requests at the same slot
and assert exactly one wins. **SC-010 (100% conflict rejection)**

## 4. Brute-force lockout (SC-011)

```bash
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:8000/api/v1/auth/login \
    -H 'Content-Type: application/json' -d '{"email":"owner@example.com","password":"wrong"}'
done
# attempts 1-9 → 401 with identical INVALID_CREDENTIALS body; attempt 10 → 429 ACCOUNT_LOCKED
curl -s -X POST localhost:8000/api/v1/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"owner@example.com","password":"correct"}' | jq -r '.error.code'   # ACCOUNT_LOCKED
curl -s "localhost:8000/api/v1/audit-events?entity_id=usr-owner&event=account_lockout" | jq '.items|length'  # ≥1
sleep 900   # or set the lockout window in .env for CI, then retry → 200
```

Counter increments must be atomic: the concurrency test in the suite issues 15 parallel
failed logins and asserts the exact threshold behaviour. **SC-011**

## 5. Dry-run publish at the intended instant (SC-003, SC-005)

```bash
# schedule a post 90 s from now, then watch dispatch latency
jq -n --arg t "$(date -u -d '+90 seconds' +%FT%TZ)" \
  '{content_item_id:"'$CID'",social_account_ids:["'$LI'"],schedule:{mode:"same",scheduled_at:$t}}' \
| curl -s -b cookies.txt -H "$CSRF" -X POST localhost:8000/api/v1/platform-posts -d @- >/dev/null
sleep 120
curl -s -b cookies.txt "localhost:8000/api/v1/platform-posts?status=published" \
  | jq '.items[0] | {status, dry_run, external_post_id}'   # published, true, "dryrun-…"
```

Worker log shows the dispatch timestamp within 15 s of `scheduled_at`; nothing outside
the workspace (no platform API call) is ever touched. **SC-003, P3/P4**

## 6. Authorization matrix (SC-007)

```bash
make test TESTPATH=apps/api/tests/test_auth_matrix.py
```

The table test logs in as viewer/editor/owner and asserts 403 for every operation the
role does not own, including direct service-layer calls (not just routes). **SC-007**

## 7. Backup, restore, retention (SC-012, SC-013, SC-008)

```bash
scripts/backup.sh --out /tmp/sos-backup-$(date +%s)     # exit 0 + manifest row counts
docker compose down -v && docker compose up -d && make db.reset   # clean environment
scripts/restore.sh --from /tmp/sos-backup-<id> && scripts/verify_restore.sh   # exit 0
make test TESTPATH=apps/api/tests/test_retention.py     # purge windows: 30 d / 90 d only
docker compose restart api worker && curl -sf localhost:8000/readyz
```

`verify_restore.sh` asserts content, schedules, audit rows and media references match
the manifest. **SC-012, SC-013, SC-008**

## 8. Restart & redeploy persistence (SC-008)

```bash
docker compose restart db api worker web && make check   # counts unchanged
docker compose down && docker compose up -d && make db.reset   # same config, same data
```

## 9. Performance smoke (SC-003, SC-006)

```bash
make seed-large && scripts/bench.sh   # 60 s, reports p50/p95 per endpoint
```

Targets: list endpoints p95 < 300 ms, dashboard p95 < 500 ms (< 1 s cold) on the
seeded database. **NFR-003 / NFR-004**

## Pass criteria for the plan

`quickstart.md` scenarios 1–7 are green **and** `make check` exits 0 — then the feature
is ready for `/speckit-converge`. Scenario output (commands + results) is pasted into
`checklists/requirements.md` as evidence; a green build alone is never the evidence (P12).
