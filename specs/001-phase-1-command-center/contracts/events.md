# Contracts: Domain Events & Adapter Payload — Social Command Center (Phase 1)

**Feature**: `001-phase-1-command-center` · **Date**: 2026-10-26 · **Phase 1 output**

Schema-versioned, additive-only (P11). Published to Redis and consumed by the worker in
Phase 1; Phase 4/5 consumers subscribe to the same topics without breaking changes.

## 1. Envelope (all events)

```jsonc
{
  "event": "post.published",          // enum, see §2
  "version": 1,                       // integer; bump only on a breaking change
  "occurred_at": "2026-10-30T08:00:12Z", // UTC, ISO-8601, producer clock
  "workspace_id": "ws-…",
  "actor_id": "usr-…",                // null for system/worker-generated events
  "entity": { "type": "platform_post", "id": "pp-…" },
  "dry_run": true,                    // true on every Phase 1 publish-family event
  "request_id": "req-…",              // correlation id, propagated from the API call
  "payload": { }                      // event-specific, see §2
}
```

Rules: consumers ignore unknown fields; producers never remove or retype a field within
`version: 1`; every event carries `workspace_id` so a consumer can shard or filter
without a lookup; no event body contains tokens, credentials or private content bodies
(P9).

## 2. Event catalogue (Phase 1)

| Event | Producer | `entity.type` | Payload |
|---|---|---|---|
| `content.created` | API | `content_item` | `{status, campaign_id, variant_count}` |
| `content.transitioned` | API | `content_item` | `{from, to, note}` |
| `content.approved` | API | `content_item` | `{from, to, approver_id}` |
| `scheduled.created` | API | `platform_post` | `{social_account_id, platform, scheduled_at}` |
| `scheduled.rescheduled` | API | `platform_post` | `{previous_scheduled_at, scheduled_at, override_reason}` |
| `post.published` | worker | `platform_post` | `{content_item_id, social_account_id, platform, external_post_id, dry_run, attempt}` |
| `post.failed` | worker | `platform_post` | `{attempt, error_code, will_retry, next_attempt_at}` |
| `notification.created` | worker/API | `notification` | `{user_id, kind, entity}` |

`post.published` and `post.failed` are emitted **after** the outbox row is committed with
its outcome, so a crash between write and publish loses neither the outcome nor the
event (research R3). `scheduled.created`/`scheduled.rescheduled` are emitted after the
spacing check commits, never before.

## 3. Job payload (queue: `publish_post`)

The worker consumes a job envelope, not the event envelope — jobs are commands,
events are facts:

```jsonc
{ "job": "publish_post", "version": 1,
  "platform_post_id": "pp-…", "workspace_id": "ws-…",
  "idempotency_key": "pp-…:ag:1",   // (platform_post_id, attempt_group); unique in jobs.idempotency_key
  "scheduled_at": "2026-10-30T08:00:00Z",
  "enqueued_at": "2026-10-30T07:59:58Z", "attempts": 0, "max_attempts": 4 }
```

Claiming uses `FOR UPDATE SKIP LOCKED`; a claimed job is deleted only after the
`platform_posts` outcome is committed (at-most-once dispatch). Retries back off
exponentially (5 s, 30 s, 2 m, 10 m; max 4 attempts), and a `failed` post after `max_attempts`
waits for the manual `POST /platform-posts/{id}/retry`.

## 4. Adapter payload & result (consumed by Phase 2)

```python
@dataclass(frozen=True)
class PlatformPostPayload:
    platform: str
    account: SocialAccount          # id, external_id, capabilities (frozen snapshot)
    body: str                       # rendered variant text
    media: list[MediaRef]           # signed, short-lived references
    card: Card | None
    scheduled_at: datetime | None   # UTC instant; None = publish now
    idempotency_key: str            # adapter must send it if the platform supports it

@dataclass(frozen=True)
class PublishResult:
    ok: bool
    external_post_id: str | None    # synthetic in Phase 1 (DryRunAdapter)
    dry_run: bool                   # True in Phase 1
    error_code: str | None          # enum: RATE_LIMITED | UNAUTHORIZED | CAPABILITY_UNAVAILABLE | PLATFORM_ERROR
    retryable: bool
    raw: dict | None                # never logged verbatim (P9)
```

`DryRunAdapter` validates the capability matrix, renders the payload it *would* send,
logs a redacted summary, and returns `ok=True`, `dry_run=True`, and
`external_post_id = f"dryrun-{idempotency_key[:8]}"`. Any capability the account does
not advertise raises `CAPABILITY_UNAVAILABLE` **before** a post row is created
(FR-004), so a Phase 1 record never claims something the matrix forbids (P14).

## 5. Topic naming

`socialos.v1.<event>` (e.g. `socialos.v1.post.published`). Consumers bind with a
wildcard on the version segment only when they tolerate unknown event types;
`post.*` fan-out must not cross workspace boundaries (every handler filters on
`workspace_id` from the envelope).
