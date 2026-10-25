# Phase 1 Spec — Social Command Center

**Status:** Draft for review · **Version:** 0.1 · **Date:** 2026-10-25
**Constitution:** [CONSTITUTION.md](CONSTITUTION.md) · **Contracts:** [API-CONTRACTS.md](API-CONTRACTS.md)
**Downstream docs:** [ARCHITECTURE.md](ARCHITECTURE.md), [ACCEPTANCE-CRITERIA.md](ACCEPTANCE-CRITERIA.md)

---

## 1. Purpose

Deliver a production-grade, self-hosted **Social Command Center**: the operator's single
surface for connecting social accounts, drafting content, organising it into campaigns,
laying it on a calendar, approving it and scheduling it for publication — on
Docker Compose, with persistent data and no AI.

This is the foundation on Phases 2–7 (platform integrations, local AI, agents,
automation, autonomous engagement, learning) are built. Phase 1 must therefore
establish the *interfaces and extension points* those phases need, without implementing
them (Constitution P13, P14).

## 2. Personas

| Persona | Need |
|---|---|
| **Operator** (owner) | See today's numbers, approve work, schedule posts, connect accounts |
| **Editor** | Create and refine content and campaigns; cannot approve or publish |
| **Viewer** | Read-only access to calendar, content and reports |

## 3. Scope

### 3.1 In scope

- Authentication and workspace-level role-based access control
- Dashboard (command centre) with activity and pending-approval surfaces
- Social account registry with connection status and capability display
- Content library: create/edit/duplicate content items and per-platform variants
- Media library (upload, preview, metadata) backed by object storage
- Campaigns: create, archive, group content, view campaign status
- Calendar: month/week views, drag-free reschedule via form, per-day filtering
- Content lifecycle state machine with approvals (approve / request changes / reject)
- Scheduling and a publish worker that dispatches to adapters (adapter stubs in Phase 1)
- Notifications for approval requests and publish outcomes
- Audit log and settings (workspace, policy defaults, members)

### 3.2 Explicitly out of scope (architecturally supported, not operationally available)

| Deferred to | Capability | What Phase 1 still provides |
|---|---|---|
| Phase 2 | Real LinkedIn/X/Facebook/Instagram publishing | `SocialPlatformAdapter` interface, capability matrix, adapter stubs + dry-run |
| Phase 2 | OAuth token exchange, comment and analytics fetching | Encrypted token columns, `Engagement`/`AnalyticsSnapshot` tables |
| Phase 3 | AI generation, rewriting, adaptation, classification | `LLMProvider` interface and `agent_runs`/`agent_tasks` tables |
| Phase 4 | Strategist/Researcher/Content/Engagement/Analytics agents, orchestrator | Permission model and agent registration hooks |
| Phase 5 | n8n workflows, scheduled research, reports | Job queue and cron-shaped worker |
| Phase 6 | Comment classification, sentiment/intent, auto-response | `Engagement` entity and policy engine fields |
| Phase 7 | Performance learning loop and memory | `AnalyticsSnapshot`, `AgentMemory` tables, event hooks |
| Later | Media generation (ComfyUI/FFmpeg) | `MediaAsset` with `generated` provenance field |

## 4. User stories and acceptance criteria

> Format: **US-xxx** — story, then AC in Given/When/Then.

**US-001 Sign in.** As an operator I sign in with email and password so that only
invited people reach the workspace.
- AC1: Given a registered user, When correct credentials are submitted, Then the session
  is established and `/dashboard` renders within one request.
- AC2: Given wrong credentials, When submitted, Then the response is `401` with a
  generic error and no account-existence leak.
- AC3: Given no session, When any `/api/v1/*` private route is requested, Then `401`
  is returned and no data leaks.

**US-002 Manage members and roles.** As an operator I invite users and assign
`owner|editor|viewer` so that approvals stay gated.
- AC1: Given an `owner`, When a member is invited, Then the invite appears as pending
  and the invited user can accept once.
- AC2: Given an `editor`, When an approval endpoint is called, Then `403` is returned.

**US-003 View the command centre.** As an operator I see today's key numbers, pending
approvals and suggested next actions so I can act without hunting.
- AC1: Given scheduled and published posts exist, When `/dashboard` loads, Then totals,
  today's posts and approval queue counts are shown from live data (≤ 1 s from cache-free
  load on a seeded database).

**US-004 Connect a social account.** As an operator I register a platform account and
see exactly what it can do.
- AC1: Given a platform adapter registered, When I add an account, Then the account is
  stored with status `connected|needs_reauth|disconnected` and its capability matrix is
  displayed.
- AC2: Given an unsupported capability, When content targeting that account is saved,
  Then validation returns `422` naming the capability (Constitution P3, P14).

**US-005 Create content and variants.** As an editor I write a base content item and
per-platform variants so that each network gets its own copy.
- AC1: Given a base item, When a variant is added for `linkedin`, Then the variant
  stores its own body, media set, character limit validation and status.
- AC2: Given a variant exceeding a platform limit, When saved, Then `422` is returned
  with the limit and current length.

**US-006 Upload media.** As an editor I upload images/video so they can be attached to
content.
- AC1: Given a valid file, When uploaded, Then a `MediaAsset` row exists with
  `storage_key`, `mime_type`, `size`, `sha256` and the blob is retrievable.
- AC2: Given an oversized or disallowed type, When uploaded, Then `413`/`422` is
  returned and nothing is stored.

**US-007 Organise with campaigns.** As an editor I create and archive campaigns and
attach content so that work is grouped.
- AC1: Given archived campaign, When its content is listed, Then it is excluded from
  default views but retrievable with an `archived=true` filter.

**US-008 Use the calendar.** As an operator I see scheduled content on a month/week
grid and reschedule from a form.
- AC1: Given posts scheduled on a date, When the month view loads, Then each appears on
  its date with platform and status.
- AC2: Given a scheduled post, When I change its time, Then `scheduled_at` updates,
  the change is audited, and a conflict with another post on the same account/time is
  surfaced.

**US-009 Move content through its lifecycle.** As an editor/operator I advance content
through the defined states so that nothing publishes unreviewed.
- AC1: Given a `DRAFT`, When submitted for review, Then state becomes `AI_REVIEWED`
  (AI skipped in Phase 1) → `HUMAN_REVIEW`, and an approval request is created.
- AC2: Given an invalid transition (e.g. `APPROVED → IDEA`), When attempted, Then
  `409` is returned with the allowed transitions.
- AC3: Given approval policy `approval_required` (default), When scheduling is attempted
  without approval, Then `422` is returned (Constitution P8).

**US-010 Approve or reject.** As an operator I approve, request changes or reject so
that only approved content can be scheduled.
- AC1: Given an approval request, When approved by an authorised user, Then state
  becomes `APPROVED` and the actor, timestamp and comment are recorded.
- AC2: When requesting changes, Then state returns to `DRAFT` with the note attached.

**US-011 Schedule and publish.** As an operator I schedule approved content and the
worker publishes it at the right time.
- AC1: Given approved content scheduled at T, When the clock passes T, Then the publish
  job runs, the adapter receives a formatted payload and the result is stored as
  `PUBLISHED` with `external_post_id` (or `FAILED` with error) and an audit event.
- AC2: Given the adapter stub (Phase 1 dry-run), Then the payload, dispatch and outcome
  are recorded exactly as a real adapter would be, clearly marked `dry_run`.
- AC3: Given a publish failure, Then retries follow the documented backoff, the post is
  marked `FAILED` after the final attempt and the operator is notified.

**US-012 Audit everything.** As an operator I can see who did what.
- AC1: Given any state change, When queried, Then `audit_events` returns actor, action,
  entity, before/after and timestamp; the list endpoint filters by entity and date.

## 5. Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-001 | Email/password auth with session revocation and role-based access | Must |
| FR-002 | Workspaces with members, roles and invitations | Must |
| FR-003 | Social account registry with connection state and capability matrix | Must |
| FR-004 | Content items with base content, variants, tags and ownership | Must |
| FR-005 | Media upload to object storage with metadata rows only in Postgres | Must |
| FR-006 | Campaign create/edit/archive and content association | Must |
| FR-007 | Calendar with month/week views and rescheduling | Must |
| FR-008 | Lifecycle state machine with enforced transitions | Must |
| FR-009 | Approval workflow (approve / request changes / reject) with policy defaults | Must |
| FR-010 | Scheduling with conflict detection per account and time | Must |
| FR-011 | Background publish worker with retry/backoff and dead-lettering | Must |
| FR-012 | Adapter interface + dry-run adapter; capability enforcement | Must |
| FR-013 | Notifications for approvals and publish outcomes (in-app + email) | Must |
| FR-014 | Audit log with filtering and export | Must |
| FR-015 | Settings: workspace, policy defaults, theme, logo | Should |
| FR-016 | Health endpoints (`/healthz`, `/readyz`) and structured logs | Must |
| FR-017 | Versioned DB migrations runnable from a clean clone | Must |
| FR-018 | Seed/demo data script for local evaluation | Should |

## 6. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-001 | Cold start from clean clone | `git clone && cp .env.example .env && docker compose up -d` works, no other manual step |
| NFR-002 | Data durability | All domain data survives `docker compose down` and container rebuild |
| NFR-003 | API latency (p95, warm, seeded) | < 300 ms for list endpoints, < 500 ms for dashboard |
| NFR-004 | Scheduling accuracy | Job fires within 15 s of its `scheduled_at` |
| NFR-005 | Security | No plaintext secrets; all private routes authenticated; input validated; errors never leak internals |
| NFR-006 | Observability | Structured JSON logs with request id; health/readiness probes |
| NFR-007 | Portability | amd64/arm64 images; runs on a single workstation |
| NFR-008 | Reproducibility | Pinned image tags and lockfiles; CI reproduces local results |

## 7. Extension points Phase 1 must establish (and not fill)

1. `SocialPlatformAdapter` ABC with the capability matrix — implemented only by a
   `DryRunAdapter`.
2. `LLMProvider` interface + config switch — no model wired.
3. Publish pipeline as discrete, observable steps
   (`validate → format → dispatch → record`) so Phase 2 adapters slot into `dispatch`.
4. Domain events (`content.approved`, `post.published`, `engagement.received`) emitted
   by the API so Phase 5 n8n and Phase 4 agents can subscribe.
5. Policy engine shape (`publishing`, `engagement`, `low_risk_replies`,
   `high_risk_content`, `politics`, `controversial`, `new_campaign`) with Phase 1
   defaults only.
6. `agent_runs`, `agent_tasks`, `agent_memories` tables created but unused.

## 8. Dependencies and risks

| Risk | Impact | Mitigation |
|---|---|---|
| Platform API permissions vary by account type (LinkedIn member vs org) | Phase 2 over-promises | Capabilities matrix is data, not code (P14); specs mark supported vs available |
| OAuth token storage | Security | Encrypted-at-rest columns from day one (P9) |
| Scheduler reliability across restarts | Missed posts | Persistent job queue with idempotent dispatch and reconciliation on boot |
| Scope creep from Phases 2–7 | Unshippable Phase 1 | Section 3.2 is binding; extension points, not implementations |

## 9. Open questions

1. Single workspace or multi-tenant from Phase 1? *(Assumed: multi-workspace schema,
   single active workspace per user.)*
2. Email delivery: local SMTP (Mailpit) in dev and pluggable provider in prod?
3. Is `viewer` allowed to see drafts of others, or only approved content?

## 10. Out-of-scope definition of "done"

Phase 1 is complete when [ACCEPTANCE-CRITERIA.md](ACCEPTANCE-CRITERIA.md) passes in CI
and on a clean machine, and the checklist for the Phase 1 feature folder is fully
evidenced.
