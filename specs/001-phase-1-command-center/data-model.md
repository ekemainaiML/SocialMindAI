# Data Model: Social Command Center (Phase 1)

**Feature**: `001-phase-1-command-center` · **Date**: 2026-10-26 · **Phase 1 output**

Source of truth for schema: `database/migrations/` (Alembic). This document is the
design those migrations must express; the contract-first reconciliation of
`scheduled_posts`/`published_posts` into `platform_posts` is recorded in
[plan.md](plan.md) §Complexity Tracking.

## Conventions

- Every table: `id uuid primary key` (UUID v4), `workspace_id uuid references workspaces(id)`
  where applicable, `created_at`/`updated_at timestamptz` (UTC, set by the database),
  soft deletion via `deleted_at timestamptz null`.
- Times are `timestamptz`; the API boundary exchanges ISO-8601 with offsets.
- All tables prefixed `smos_` in the database; CamelCase names here.
- Enums are Postgres `enum` types with a `CHECK` fallback so migrations stay reversible.
- Every workspace-scoped query filters by `workspace_id` taken from the session, never
  from the request body (security baseline, P7).

## Table inventory

| Table | Phase 1 use |
|---|---|
| `users`, `workspaces`, `memberships`, `invitations` | identity, RBAC, invites |
| `auth_attempts` | brute-force counters and lockouts (new in Phase 1) |
| `social_accounts`, `platform_tokens` | registry + encrypted tokens (exchange in Phase 2) |
| `campaigns`, `content_items`, `content_variants`, `content_variant_media` | content domain |
| `media_assets` | object-storage metadata only (P9) |
| `platform_posts` | scheduling, publishing, retries, outcome (merged entity, R1) |
| `jobs` | durable outbox for the worker (R2) |
| `approval_requests` | human gate |
| `notifications`, `audit_events`, `settings` | notifications, audit, policy |
| `comments`, `engagements`, `analytics_snapshots` | created empty; Phase 2/6/7 fill |
| `agent_runs`, `agent_tasks`, `agent_memories` | created empty; Phase 4 fills |
| `automation_rules` | created empty; Phase 5 fills |

## Entities

### User
`id`, `email` (citext, unique), `password_hash` (Argon2id), `display_name`,
`is_active`, `last_login_at`, timestamps.
Validation: email normalised and lowercased at write; password ≥ 12 characters.

### Workspace
`id`, `name`, `timezone` (IANA identifier, validated on write — FR-026),
`policy` (jsonb: `approval_required` default `true`, `spacing_minutes` default `60`,
`lockout_threshold` default `10`, `lockout_window_minutes` default `15`,
`lockout_duration_minutes` default `15`, `retention_soft_delete_days` default `30,
`retention_notification_days` default `90`), timestamps.
Validation: a missing or invalid timezone is rejected, never defaulted to server time.

### Membership
`workspace_id`, `user_id`, `role` (`owner`|`editor`|`viewer`), timestamps.
Constraint: unique `(workspace_id, user_id)`. Invariant: a workspace always retains at
least one owner (enforced in the service layer, `409` otherwise) — US-002 scenario 4.

### Invitation
`workspace_id`, `email`, `role`, `token_hash`, `invited_by`, `accepted_at`,
`expires_at`. Constraint: one acceptance per invitation; a second acceptance is `409`.

### AuthAttempt
`bucket` (`identity:<email>` | `ip:<address>`), `window_start`, `failures int`,
`locked_until timestamptz null`, `updated_at`. Constraint: unique `(bucket, window_start)`.
Rules: incremented with an atomic upsert inside the authentication transaction; a
successful sign-in resets the identity bucket; `locked_until` set when
`failures >= policy.lockout_threshold` inside the rolling window (FR-001, R4).

### SocialAccount
`workspace_id`, `platform` (`linkedin`|`x`|`facebook`|`instagram`), `account_name`,
`external_id`, `status` (`connected`|`needs_reauth`|`disconnected`),
`capabilities` (jsonb: `publish_text`, `publish_image`, `publish_video`,
`read_comments`, `reply_comments`, `read_analytics`), `last_sync_at`, timestamps,
`deleted_at`. Constraint: unique `(workspace_id, platform, external_id)`.
Capabilities are seeded from the platform definition at registration and are data, not
code (FR-004, R6).

### PlatformToken
`social_account_id` (1—1), `encrypted_access_token`, `encrypted_refresh_token`,
`token_expires_at`, `scopes`, `last_refreshed_at`. AES-GCM under `ENCRYPTION_KEY` (P9).

### Campaign
`workspace_id`, `name`, `description`, `status` (`active`|`archived`), timestamps,
`deleted_at`. Archived campaigns are excluded from default listings unless
`?archived=true` (FR-007).

### ContentItem
`workspace_id`, `campaign_id` (nullable), `title`, `base_body`, `status` (lifecycle,
see below), `tags` (text[]), `owner_id`, `submitted_by`, `embedding` (pgvector,
nullable, unused in Phase 1), timestamps, `deleted_at`, `version int` (optimistic
locking for `PATCH` via `If-Match`).

### ContentVariant
`content_item_id`, `platform`, `body`, `card` (jsonb nullable),
`platform_limit int` (chars, seeded per platform), timestamps.
Validation: `body` length ≤ `platform_limit`, else `422` naming limit and current
length (US-004 scenario 2). Constraint: unique `(content_item_id, platform)`.

### ContentVariantMedia
`content_variant_id`, `media_asset_id`, `position int`. M:N join (US-005).

### MediaAsset
`workspace_id`, `storage_key`, `original_name`, `mime_type`, `size_bytes`,
`sha256`, `provenance` (`uploaded`|`generated`, Phase 1 always `uploaded`),
`uploaded_by`, timestamps, `deleted_at`. Constraint: unique `(workspace_id, sha256)`
so re-uploads reuse one object. Blobs live in MinIO only (P9).

### PlatformPost *(the merged entity — R1)*
`workspace_id`, `content_item_id`, `content_variant_id`, `social_account_id`,
`scheduled_at timestamptz`, `status`
(`scheduled`|`publishing`|`published`|`failed`|`cancelled`),
`attempts int`, `max_attempts int` (default 4), `idempotency_key text` (unique),
`override_reason text null`, `override_by uuid null`,
`published_at timestamptz null`, `external_post_id text null`, `dry_run boolean`,
`last_error text null`, `cancelled_at`, timestamps.
Rules:
- Created only from `APPROVED` content, else `422 APPROVAL_REQUIRED` (FR-010).
- Fan-out creates N rows in one transaction; each row is edited, rescheduled and
  retried independently (FR-022).
- Status transitions: `scheduled → publishing → published | failed`;
  `failed → publishing` (manual retry); `scheduled → cancelled`.
  A transition out of `published` is impossible (at-most-once, FR-012).
- `idempotency_key = f"{content_item_id}:{social_account_id}:{scheduled_at}"` for the
  first attempt group; retries extend the attempt group rather than creating a new
  post (R2).
- Spacing: a row can only be created or rescheduled when no other `scheduled` row on
  the same account lies within `±policy.spacing_minutes`, unless `override_reason` is
  present and the actor is an owner — then both rows, the account, timestamps and the
  reason are written to `audit_events` (FR-023, SC-010).
- `dry_run` is `true` for every Phase 1 post (P14).

### Job *(durable outbox)*
`id`, `kind` (`publish_post`|`notify`|`purge_retention`|`reconcile_schedule`),
`payload` (jsonb), `run_at timestamptz`, `status` (`pending`|`claimed`|`done`|`failed`),
`claimed_by`, `claimed_at`, `attempts`, `idempotency_key` (unique), timestamps.
Claimed with `FOR UPDATE SKIP LOCKED`; a claimed job whose worker died is reclaimable
after a lease timeout (R2).

### ApprovalRequest
`workspace_id`, `content_item_id`, `requested_by`, `decided_by` (nullable),
`status` (`pending`|`approved`|`changes_requested`|`rejected`), `note`,
`decided_at`. Constraint: one open request per content item.
Separation of duties: the submitter cannot decide their own request — `403`
(US-009 scenario 4).

### Notification
`workspace_id`, `user_id`, `kind` (`approval_requested`|`publish_succeeded`|
`publish_failed`|`account_needs_reauth`), `entity_type`, `entity_id`, `payload`
(jsonb), `read_at` (nullable), `emailed_at` (nullable), `created_at`.
Deduplication: unique `(user_id, kind, entity_id)` so retries never double-notify
(FR-014).

### AuditEvent
`workspace_id`, `actor_type` (`user`|`system`|`agent`), `actor_id`, `action`,
`entity_type`, `entity_id`, `before` (jsonb), `after` (jsonb), `request_id`,
`created_at`. Append-only: no UPDATE or DELETE path exists (P10, SC-004, retention
"forever").

### Settings
`workspace_id` (unique), `policy` overrides, `theme`, `logo_media_id`, timestamps.
Policy reads merge workspace overrides over constitution defaults.

### Empty tables for later phases
`comments`, `engagements` (Phase 2/6), `analytics_snapshots` (Phase 2/7),
`agent_runs`, `agent_tasks`, `agent_memories` (Phase 4/7, `embedding pgvector`),
`automation_rules` (Phase 5). Created in Phase 1 migrations so later phases add
columns, not tables (P13).

## Relationships

```
workspaces 1—* memberships *—1 users
workspaces 1—* invitations
workspaces 1—* social_accounts 1—1 platform_tokens
workspaces 1—* campaigns 1—* content_items 1—* content_variants
content_variants *—* media_assets (via content_variant_media)
content_items 1—* approval_requests
content_items 1—* platform_posts *—1 social_accounts
platform_posts 1—* jobs (one per attempt group)
platform_posts → post.published / post.failed events
* → audit_events (polymorphic: entity_type, entity_id, actor_type, actor_id)
notifications *—1 users
auth_attempts → workspace-less (identity and IP buckets)
```

## Content lifecycle state machine

`content_items.status` — DB CHECK constraint plus an application transition table;
an invalid transition returns `409` naming the allowed ones (FR-009).

```
IDEA → RESEARCHED → DRAFT → AI_REVIEWED → HUMAN_REVIEW → APPROVED → SCHEDULED
     → PUBLISHED → MONITORING → ANALYZED → LEARNING
                     ↘ CHANGES_REQUESTED → DRAFT      ↘ FAILED → HUMAN_REVIEW
```

Rules:
- Submitting for review executes `DRAFT → AI_REVIEWED → HUMAN_REVIEW` as one
  pass-through with a note that no model ran, and creates the `ApprovalRequest`
  (FR-009, R8).
- `APPROVED → SCHEDULED` requires at least one `platform_posts` row (FR-010).
- Editing an item in `HUMAN_REVIEW` or later is refused until it returns to `DRAFT`
  (US-008 scenario 3).
- `PUBLISHED → MONITORING → ANALYZED → LEARNING` are written by the API as reserved
  transitions; only `PUBLISHED` is reachable in Phase 1.

## Validation and integrity rules derived from the spec

| Rule | Source |
|---|---|
| Capability must exist on the target account before scheduling, else `422` naming it | FR-004, US-003 |
| Variant body ≤ platform limit, else `422` with limit and length | FR-005, US-004 |
| Scheduling unapproved content → `422 APPROVAL_REQUIRED` | FR-010, US-009 |
| Invalid lifecycle transition → `409` with allowed list | FR-009, US-008 |
| Same-account spacing conflict → `409` naming the clash; owner override needs a reason | FR-023, SC-010 |
| Optimistic locking on content edits (`If-Match: version`) → `409` on mismatch | US-012 edge cases |
| Media upload: MIME allow-list, ≤ 50 MB, else `422`/`413`, nothing stored | FR-006, US-005 |
| Deleting media referenced by a scheduled post → `409` | §2.4 of the contract |
| Every mutation writes exactly one `audit_event` with before/after | FR-015, SC-004 |
| Retention purge touches only `deleted_at` older than 30 days and notifications older than 90 days | FR-024, SC-013 |

## Retention

| Data | Rule |
|---|---|
| Audit events, content, platform posts | Indefinite |
| Soft-deleted content | Purged after 30 days |
| Notifications | Purged after 90 days |
| `auth_attempts` rows | Rolled window; rows older than 24 h purged |
| Backups | Operator-configurable snapshot retention, documented |
