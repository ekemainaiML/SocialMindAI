# Social OS — API Contracts (Phase 1)

**Status:** Draft · **Version:** 0.1 · **Date:** 2026-10-25 · **Constitution:** P11 (contract-first)
Machine-readable source of truth: `apps/api/openapi.json` generated from this contract;
drift fails CI. Companion: [PHASE-1-SPEC.md](PHASE-1-SPEC.md), [ARCHITECTURE.md](ARCHITECTURE.md)

---

## 1. Conventions

| Aspect | Rule |
|---|---|
| Base path | `/api/v1` |
| Transport | JSON over HTTPS; `Content-Type: application/json` (uploads: `multipart/form-data`) |
| Auth | `Cookie: session=…` (`HttpOnly`, `SameSite=Lax`) + `X-CSRF-Token` for mutations |
| Errors | `{"error":{"code","message","details","request_id"}}` |
| Success list | `{"items":[…],"page":{"limit","offset,"total"}}` (use `cursor` for feeds) |
| IDs | UUID v4 strings |
| Times | ISO-8601 UTC (`2026-08-19T10:00:00Z`) |
| Idempotency | `Idempotency-Key` header honoured on POST that create jobs |
| Versioning | path versioning; additive changes only within `v1`, breaking ⇒ `v2` |
| Pagination defaults | `limit=50`, `max=200` |

Status codes: `200/201/202` success · `400` malformed · `401` unauthenticated ·
`403` forbidden (role/capability) · `404` not found *in this workspace* ·
`409` state conflict (illegal transition, duplicate) · `413` too large ·
`422` validation (field-level `details`) · `429` rate limited · `500` internal.

## 2. Endpoint map

### 2.1 Identity & workspace

| Method | Path | Role | Notes |
|---|---|---|---|
| POST | `/auth/register` | public | creates owner + workspace |
| POST | `/auth/login` | public | sets session, `429` after 5 failures/15 min |
| POST | `/auth/logout` | authenticated | revokes session |
| GET | `/auth/me` | authenticated | user, workspace, roles, capabilities |
| POST | `/auth/password` | authenticated | change password, revokes other sessions |
| GET/POST | `/workspaces/{id}/members` | owner | list, invite |
| PATCH/DELETE | `/workspaces/{id}/members/{user_id}` | owner | role change, removal |

### 2.2 Social accounts

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/social-accounts` | viewer+ | includes `capabilities` matrix |
| POST | `/social-accounts` | owner | `{platform, account_name, external_id}` → `connected` or `needs_reauth` |
| GET | `/social-accounts/{id}` | viewer+ | status, capabilities, last_sync |
| POST | `/social-accounts/{id}/refresh` | owner | re-authorise (Phase 2: OAuth) |
| DELETE | `/social-accounts/{id}` | owner | soft-delete, revokes stored token |
| GET | `/platforms` | viewer+ | supported platforms and their capability matrices |

### 2.3 Content

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/content` | viewer+ | filters: `status`, `campaign_id`, `q`, `tag`, `archived` |
| POST | `/content` | editor+ | creates `IDEA`/`DRAFT` with base content |
| GET | `/content/{id}` | viewer+ | item + variants + media + history |
| PATCH | `/content/{id}` | editor+ | optimistic locking via `If-Match: <updated_at>` |
| DELETE | `/content/{id}` | editor+ | soft delete |
| POST | `/content/{id}/transition` | editor+ | `{to_state, note}` → `200` or `409` with allowed transitions |
| GET/POST | `/content/{id}/variants` | editor+ | per-platform variant; `422` if platform capability unsupported |
| PUT | `/content/variants/{id}` | editor+ | body, media, scheduled preference |

```jsonc
// POST /api/v1/content/5f0e…/variants  → 201
{
  "platform": "linkedin",
  "body": "AI engineering in African manufacturing: what actually ships.",
  "media_asset_ids": ["0a1b…"],
  "card": {"title": "AI in manufacturing", "description": "Field notes"}
}
```

### 2.4 Media

| Method | Path | Role | Notes |
|---|---|---|---|
| POST | `/media` | editor+ | multipart; returns `MediaAsset`; `413` over 50 MB, `422` bad MIME |
| GET | `/media` | viewer+ | paginated library |
| GET | `/media/{id}/url` | viewer+ | short-lived signed URL |
| DELETE | `/media/{id}` | editor+ | refuses if referenced by scheduled content (`409`) |

### 2.5 Campaigns & calendar

| Method | Path | Role | Notes |
|---|---|---|---|
| GET/POST | `/campaigns` | viewer+/editor+ | `?archived=true` includes archived |
| PATCH/DELETE | `/campaigns/{id}` | editor+ | archive is the default delete |
| GET | `/calendar` | viewer+ | `?from=YYYY-MM-DD&to=…&account_id=…` → grouped by date |
| GET | `/scheduled-posts` | viewer+ | list with filters |
| POST | `/scheduled-posts` | editor+ | `{variant_id, account_id, scheduled_at}` → `409` on time conflict, `422` if not `APPROVED` |
| PATCH | `/scheduled-posts/{id}` | editor+ | reschedule; audited |

```jsonc
// POST /api/v1/scheduled-posts → 201
{ "content_variant_id": "…", "social_account_id": "…",
  "scheduled_at": "2026-08-19T10:00:00Z" }

// 422 (policy default, PHASE-1-SPEC US-009 AC3)
{ "error": { "code": "APPROVAL_REQUIRED",
  "message": "Content must be APPROVED before scheduling.",
  "details": {"current_status": "HUMAN_REVIEW"}, "request_id": "…" } }
```

### 2.6 Approvals

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/approvals` | owner, editor (own) | `?status=pending` |
| POST | `/approvals/{id}/approve` | owner | → item `APPROVED`, audit event |
| POST | `/approvals/{id}/request-changes` | owner | `{note}` → item `DRAFT` |
| POST | `/approvals/{id}/reject` | owner | → item `DRAFT`, reason required |

### 2.7 Publishing, notifications, audit, settings

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/published-posts` | viewer+ | includes `dry_run`, `external_post_id`, errors |
| POST | `/published-posts/{id}/retry` | owner | manual re-dispatch |
| GET | `/notifications` | authenticated | `?unread=true`; `POST /{id}/read` |
| GET | `/audit-events` | owner | `?entity_type=&entity_id=&from=&to=` |
| GET/PUT | `/settings/policy` | owner | approval policy (defaults from PHASE-1-SPEC) |
| GET | `/dashboard/summary` | viewer+ | counts, today's posts, approval queue, recommendations[] (static in Phase 1) |
| GET | `/healthz`, `/readyz` | public | liveness / readiness |

## 3. Domain event contracts (extension points, P3/P5)

Emitted to Redis for Phase 4/5 consumers; schema is versioned, additive only.

```jsonc
{ "event": "post.published", "version": 1, "occurred_at": "2026-08-19T10:00:12Z",
  "workspace_id": "…", "content_id": "…", "variant_id": "…",
  "social_account_id": "…", "platform": "linkedin", "dry_run": true,
  "external_post_id": null, "request_id": "…" }
```
Events in Phase 1: `content.created`, `content.transitioned`, `content.approved`,
`scheduled.created`, `scheduled.rescheduled`, `post.published`, `post.failed`,
`notification.created`.

## 4. Adapter interface (contract consumed by Phase 2)

```python
class SocialPlatformAdapter(Protocol):
    platform: str
    def capabilities(self) -> Capabilities: ...          # authoritative matrix
    async def authenticate(self, credentials) -> AuthResult: ...
    async def get_profile(self, account) -> Profile: ...
    async def create_post(self, payload: PlatformPost) -> PublishResult: ...
    async def upload_media(self, account, media) -> MediaRef: ...
    async def get_comments(self, post_id) -> list[Comment]: ...
    async def reply_to_comment(self, comment_id, text) -> ReplyResult: ...
    async def get_metrics(self, post_id) -> Metrics: ...
    async def delete_post(self, post_id) -> None: ...
```
Phase 1 ships `DryRunAdapter` only: it validates capabilities, logs the payload as it
*would* dispatch, and returns a synthetic `external_post_id` marked `dry_run=true`.
