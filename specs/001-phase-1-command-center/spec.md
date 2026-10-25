# Feature Specification: Social Command Center (Phase 1)

**Feature Branch**: `001-phase-1-command-center`

**Created**: 2026-10-25

**Status**: Draft

**Input**: User description: "Full Phase 1 of Social OS — the Social Command Center: sign-in, members and roles, dashboard, social account registry, content library with per-platform variants, media, campaigns, calendar, content lifecycle with approvals, scheduling and a publish pipeline, notifications and audit log."

**Constitution**: `.specify/memory/constitution.md` (P1–P14). Every section below must
comply with it; violations are recorded in the compliance table at the bottom, not
silently accepted.

**Baseline**: this feature covers all of [docs/PHASE-1-SPEC.md](../../docs/PHASE-1-SPEC.md),
which remains the human-readable baseline. Where wording differs, this spec governs.

## Phase Gate *(Social OS preset — mandatory)*

- **Phase**: 1 — command center.
- **Constitution check**:
  - **P3 (adapters only for platform logic)** — PASS. Platform behaviour in this feature
    is reached solely through the adapter interface and its capability matrix; no
    platform-specific rules in core code.
  - **P5 (FastAPI owns the logic)** — PASS. All business rules described here live in
    the API/service layer; no workflow tool is required to run Phase 1.
  - **P8 (approval default)** — PASS. Scheduling requires approval by default; the spec
    makes the default explicit and lets policy relax it only deliberately.
  - **P9 (no secrets in tree)** — PASS. Credentials and storage keys are supplied
    through environment/secret configuration; OAuth material is stored encrypted.
  - **P11 (contract first)** — PASS. Every requirement below maps to an endpoint in
    `docs/API-CONTRACTS.md`; contract changes precede code.
  - **P13 (SDD loop followed)** — PASS. This spec, its checklist, plan and tasks are
    written and evidenced before implementation.
- **Supported vs Available (P14)**:

| Capability this feature touches | Marking | Permission source / note |
|---|---|---|
| Store a social account and its capability matrix | `available` | Local registry only; no third-party call |
| Publish to a real platform (X, LinkedIn, Facebook, Instagram) | `supported-only` | Adapter interface + dry-run adapter only; live posting arrives with Phase 2 and each platform's own API approval |
| Read comments / analytics from a platform | `supported-only` | Tables and event hooks reserved; no read path until Phase 2 |
| In-app + email notifications | `available` | Local SMTP in dev; pluggable provider in production |
| AI drafting, review, classification | `supported-only` | `AI_REVIEWED` is traversed without a model; model interface reserved for Phase 3 |
| Agents, automation, memory | `supported-only` | Tables created, unused; extension points only |

## User Scenarios & Testing *(mandatory)*

Each story is independently testable; implementing a single story still yields a usable
system. Priority: P1 = most critical.

### User Story 1 - Sign in to the workspace (Priority: P1)

An operator signs in with email and password and reaches the command center; anyone
without a valid session is refused.

**Why this priority**: nothing else is reachable or attributable without identity — it
is the gate for every other story.

**Independent Test**: With a registered user, submit credentials and confirm the session
and landing page; with no session, confirm every private route refuses.

**Acceptance Scenarios**:

1. **Given** a registered user, **When** correct credentials are submitted, **Then** a
   session is established and the command center renders.
2. **Given** wrong credentials, **When** submitted, **Then** the attempt is refused with
   a generic error that reveals nothing about whether the account exists.
3. **Given** no session, **When** any private route or API path is requested, **Then**
   the request is refused with no data returned.
4. **Given** an expired or revoked session, **When** a request is made, **Then** it is
   refused and the user is returned to sign-in.

---

### User Story 2 - Invite members and assign roles (Priority: P2)

The operator invites people as `owner`, `editor` or `viewer` so that approvals and
publishing stay gated to the right people.

**Why this priority**: approval gating and least privilege (P7) depend on roles existing.

**Independent Test**: Invite a user, accept once, then attempt a privileged action as
`editor` and `viewer` and confirm refusal.

**Acceptance Scenarios**:

1. **Given** an owner, **When** a member is invited, **Then** the invitation appears as
   pending and can be accepted exactly once.
2. **Given** an editor, **When** an approval or publish action is attempted, **Then**
   it is refused with a clear permission error.
3. **Given** a viewer, **When** a write action is attempted, **Then** it is refused, but
   read access to permitted views still works.
4. **Given** the last owner, **When** a change would leave the workspace ownerless,
   **Then** it is refused.

---

### User Story 3 - Connect a social account and see what it can do (Priority: P2)

The operator registers a platform account and sees its connection status and the exact
set of things that account may do.

**Why this priority**: content can only be targeted at known accounts, and capability
enforcement (P3, P14) is meaningless without the registry.

**Independent Test**: Add an account, inspect its capability display, then attempt to
target an unsupported capability and confirm rejection.

**Acceptance Scenarios**:

1. **Given** a registered platform, **When** an account is added, **Then** it is stored
   with a status of `connected`, `needs_reauth` or `disconnected` and its capability
   matrix is displayed.
2. **Given** an account whose capabilities exclude a feature, **When** content targeting
   it is saved, **Then** the save is rejected and the message names the capability that
   is unavailable.
3. **Given** an account marked `needs_reauth`, **When** scheduling against it is
   attempted, **Then** the attempt is refused with a reconnection prompt.

---

### User Story 4 - Write content with per-platform variants (Priority: P2)

An editor writes one base content item and, for each target network, a variant with its
own wording, media and limits.

**Why this priority**: content is the core object every later story operates on.

**Independent Test**: Create a base item, add a variant, save an over-limit variant and
confirm rejection with the limit stated.

**Acceptance Scenarios**:

1. **Given** a base item, **When** a variant for a network is added, **Then** it holds
   its own body, media set, per-platform limit validation and status.
2. **Given** a variant exceeding its platform limit, **When** saved, **Then** the save is
   rejected stating the limit and the current length.
3. **Given** an item with variants, **When** duplicated, **Then** a new draft item is
   created carrying copies, not references, of its variants.

---

### User Story 5 - Upload media (Priority: P3)

An editor uploads images and video and attaches them to content.

**Why this priority**: media is required by most posts but content without media is
still usable.

**Independent Test**: Upload a valid file and retrieve it; upload an oversized or
unsupported file and confirm nothing is stored.

**Acceptance Scenarios**:

1. **Given** a valid file within the allowed type and size, **When** uploaded, **Then**
   it is retrievable and its metadata (type, size, checksum) is recorded.
2. **Given** an oversized or disallowed file, **When** uploaded, **Then** it is rejected
   and no stored artifact remains.
3. **Given** an uploaded asset, **When** it is referenced by no content, **Then** it is
   still listed in the media library and can be deleted.

---

### User Story 6 - Group content into campaigns (Priority: P3)

An editor creates campaigns, attaches content and archives finished ones.

**Why this priority**: organisation improves throughput but single items work without it.

**Independent Test**: Create a campaign, attach content, archive it and confirm default
views hide it while a filter still reveals it.

**Acceptance Scenarios**:

1. **Given** a campaign, **When** content is attached, **Then** the content reports its
   campaign in listings and detail views.
2. **Given** an archived campaign, **When** default listings load, **Then** it and its
   content are excluded unless an "archived" filter is applied.

---

### User Story 7 - See scheduled work on a calendar (Priority: P3)

The operator views scheduled content on a month or week grid and reschedules it through
a form.

**Why this priority**: a faster alternative to list views; the underlying scheduling
works without it.

**Independent Test**: Schedule posts, view the grid, change one post's time and confirm
the change and its conflict warning.

**Acceptance Scenarios**:

1. **Given** posts scheduled across dates, **When** the month or week view loads,
   **Then** each appears on its date with its platform and status.
2. **Given** a scheduled post, **When** its time is changed, **Then** the new time is
   stored, recorded in the audit history, and a clash with another post on the same
   account and time is surfaced before confirmation.
3. **Given** a scheduled post in a timezone other than the workspace default, **When**
   the grid renders, **Then** it appears on the correct local day.

---

### User Story 8 - Move content through its lifecycle (Priority: P2)

An editor advances content through the defined states; illegal jumps are refused so
nothing skips review.

**Why this priority**: the state machine is the auditability guarantee (P10) and the
precondition for approval.

**Independent Test**: Attempt each legal transition, then attempt an illegal one and
confirm rejection listing what is allowed.

**Acceptance Scenarios**:

1. **Given** a draft, **When** it is submitted for review, **Then** it moves to human
   review (passing through the model-review step, which is skipped in Phase 1) and an
   approval request is created.
2. **Given** any state, **When** an invalid transition is attempted, **Then** it is
   rejected and the response names the currently allowed transitions.
3. **Given** content in review, **When** an edit is made without returning it to draft,
   **Then** the edit is refused so reviewed text cannot change silently.

---

### User Story 9 - Approve, request changes or reject (Priority: P2)

The operator approves, requests changes or rejects; only approved content can be
scheduled.

**Why this priority**: this is the human gate the constitution requires by default (P8).

**Independent Test**: Approve a request as an authorised user, request changes as
another, and confirm an unapproved item cannot be scheduled.

**Acceptance Scenarios**:

1. **Given** an approval request, **When** an authorised user approves it, **Then** the
   content becomes approved and the actor, time and comment are recorded.
2. **Given** an approval request, **When** changes are requested, **Then** the content
   returns to draft carrying the reviewer's note.
3. **Given** approval-required policy (the default), **When** scheduling is attempted on
   unapproved content, **Then** the attempt is rejected with a message naming the
   missing approval.
4. **Given** the same user who submitted the content, **When** they attempt to approve
   it, **Then** the attempt is refused under the separation-of-duties policy.

---

### User Story 10 - Schedule and publish (Priority: P1)

The operator schedules approved content; at the appointed time the system publishes it
through the adapter and records exactly what happened.

**Why this priority**: publication is the outcome the whole product exists to produce —
without it there is no command center, only a drafts folder.

**Independent Test**: Schedule a post, let it come due, and confirm the dispatch, the
recorded outcome and the audit entry; then force a failure and confirm retry and
notification.

**Acceptance Scenarios**:

1. **Given** approved content scheduled at time T, **When** the clock passes T, **Then**
   the publish job runs, the adapter receives a fully formatted payload, and the outcome
   is stored as published with an external reference, or failed with its error, plus an
   audit entry.
2. **Given** the Phase 1 dry-run adapter, **When** the job runs, **Then** the payload,
   dispatch and outcome are recorded exactly as a real adapter would be, and are clearly
   marked as a dry run.
3. **Given** a failed publish, **When** retries are due, **Then** they follow the
   documented backoff, and after the final attempt the post is marked failed and the
   operator is notified.
4. **Given** a post already marked published, **When** its job runs again after a
   restart, **Then** it is not published a second time.
5. **Given** a job whose time passed while the system was down, **When** the system
   recovers, **Then** overdue jobs are reconciled rather than lost or fired twice.

---

### User Story 11 - Get notified (Priority: P3)

The operator receives in-app and email notifications for approval requests and publish
outcomes.

**Why this priority**: work can be driven by watching the app; notifications remove the
need to poll.

**Independent Test**: Trigger an approval request and a publish failure and confirm both
arrive in-app and by email, then confirm they can be marked read.

**Acceptance Scenarios**:

1. **Given** a new approval request, **When** it is created, **Then** each eligible
   approver receives an in-app and an email notification once — not once per retry.
2. **Given** a publish failure, **When** the final attempt fails, **Then** the
   author and operator are notified with the failure reason.
3. **Given** a notification, **When** it is marked read, **Then** it leaves the unread
   count but remains retrievable.

---

### User Story 12 - Read the command center dashboard (Priority: P3)

The operator lands on a dashboard showing today's numbers, the approval queue and
suggested next actions.

**Why this priority**: it summarises other stories; it is valuable only once data
exists.

**Independent Test**: With seeded data, load the dashboard and confirm every number
matches a direct query of the underlying records.

**Acceptance Scenarios**:

1. **Given** scheduled, published and pending-approval records, **When** the dashboard
   loads, **Then** totals, today's items and queue counts are shown from live data.
2. **Given** nothing scheduled, **When** the dashboard loads, **Then** empty states guide
   the user to the next useful action instead of showing zeros alone.

---

### User Story 13 - Audit who did what (Priority: P2)

The operator can trace every state change to a person, a time and a before/after value.

**Why this priority**: without attribution, approvals and publishing are unauditable,
which the constitution forbids.

**Independent Test**: Perform each mutating action, then query the audit trail filtered
by entity and date and confirm every action appears.

**Acceptance Scenarios**:

1. **Given** any state change, **When** the audit trail is queried, **Then** it returns
   actor, action, entity, before and after values, and timestamp.
2. **Given** a read action, **When** it completes, **Then** no audit row is written for
   it, keeping the trail focused on changes.

---

### Edge Cases

- The worker or host is offline when a scheduled post comes due → job survives and is
  reconciled on recovery without double-publishing.
- A publish fails partway (network accepted the call but the response was lost) →
  retry logic must not create a duplicate post on the platform.
- A social account is disconnected, revoked or rate-limited between scheduling and
  publish time → the job fails closed with a clear reason and a notification rather
  than attempting a partial publish.
- Two users approve, reject or reschedule the same item concurrently → last-write-wins
  is prevented; the conflict is surfaced and both actions are audited.
- Two posts target the same account at the same minute → a conflict is surfaced before
  confirmation, and configurable spacing rules may block it.
- An approval is left pending indefinitely → it ages visibly on the dashboard and does
  not silently expire.
- Content is edited after approval → refused; it must return to draft first.
- Daylight-saving transitions and workspace/user timezone differences → a post scheduled
  for a local wall-clock time publishes at the intended instant, not an hour off.
- Media upload interrupted mid-transfer → no partial asset is listed or usable.
- A user's session expires while they are editing → their work is not silently
  submitted under a dead session, and they are returned to sign-in.
- A user role is changed while they hold an open approval → the action is evaluated
  against their current role at submission time, not at page load.
- An account's capability set shrinks after a platform policy change → content targeting
  a now-unavailable capability is blocked before scheduling, not at publish time.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST authenticate users by email and password, maintain
  revocable sessions, and refuse every private route without one.
- **FR-002**: The system MUST support workspaces containing members with the roles
  `owner`, `editor` and `viewer`, plus invitations that can be accepted exactly once.
- **FR-003**: The system MUST enforce those roles on every mutating operation, with
  `owner` for members, settings, approvals and publishing policy; `editor` for content,
  media and campaigns; `viewer` for read-only access.
- **FR-004**: The system MUST maintain a social account registry recording the platform,
  handle, connection status and the account's capability matrix, and MUST reject any
  content targeting a capability the account does not have.
- **FR-005**: The system MUST store content as a base item with per-platform variants,
  each variant holding its own body, media references and platform limit validation.
- **FR-006**: The system MUST accept media uploads with type and size validation,
  record their metadata, and serve them back intact; the database MUST hold metadata
  only.
- **FR-007**: The system MUST let users create, edit, duplicate and archive campaigns
  and associate content with them, excluding archived items from default views.
- **FR-008**: The system MUST expose calendar views (month and week) of scheduled
  content and support rescheduling through a validated form with conflict detection.
- **FR-009**: The system MUST enforce the content lifecycle
  `IDEA → RESEARCHED → DRAFT → AI_REVIEWED → HUMAN_REVIEW → APPROVED → SCHEDULED →
  PUBLISHED → MONITORING → ANALYZED → LEARNING`, rejecting invalid transitions and
  naming the allowed ones; the model-review step is traversed without a model in Phase 1.
- **FR-010**: The system MUST provide an approval workflow (approve, request changes,
  reject) with approval required by default before scheduling, separation of duties
  between submitter and approver, and actor/timestamp/comment recorded on every decision.
- **FR-011**: The system MUST schedule approved content and dispatch it at the
  appointed time through the adapter interface, recording the outcome, retrying with
  documented backoff, and marking the post failed after the final attempt.
- **FR-012**: The system MUST guarantee at-most-once publication per scheduled post
  across restarts and retries, and MUST reconcile overdue jobs on recovery.
- **FR-013**: The system MUST reach platforms only through the adapter interface and
  its capability matrix; Phase 1 ships a dry-run adapter that records payloads and
  outcomes exactly as a real adapter would, marked as a dry run.
- **FR-014**: The system MUST notify users in-app and by email of approval requests and
  publish outcomes, without duplicate notifications for retried work.
- **FR-015**: The system MUST record an audit event for every state change with actor,
  action, entity, before/after values and timestamp, and MUST expose filtered queries
  of that trail.
- **FR-016**: The system MUST provide health and readiness endpoints and structured
  logs carrying a request identifier.
- **FR-017**: The system MUST apply versioned schema migrations that run from a clean
  checkout without manual steps.
- **FR-018**: The system MUST persist all domain data across container restarts and
  rebuilds.
- **FR-019**: The system MUST keep credentials, storage keys and platform tokens out of
  the repository and encrypt stored platform tokens at rest.
- **FR-020**: The system MUST provide seed/demo data so a fresh install can be
  evaluated without manual data entry.
- **FR-021**: The system MUST emit domain events (`content.approved`,
  `post.published`, `engagement.received`) for later phases to subscribe to, without
  any subscriber in Phase 1.

### Key Entities

- **User**: identity and credentials; member of zero or more workspaces.
- **Workspace**: the tenant boundary; owns all other records, settings and policy
  defaults.
- **Membership**: the link between a user and a workspace carrying their role
  (`owner`/`editor`/`viewer`).
- **SocialAccount**: a connected platform presence — platform, handle, connection
  status, capability matrix, encrypted token material.
- **Campaign**: a grouping for content with lifecycle `active`/`archived`.
- **ContentItem**: the canonical piece of content — base body, state, ownership,
  tags, campaign link.
- **ContentVariant**: a per-platform rendering of an item with its own body, media
  and limit validation.
- **MediaAsset**: an uploaded file's metadata (type, size, checksum, storage key);
  the bytes live in object storage.
- **ScheduledPost**: the intent to publish a variant to an account at a time, with
  status, attempt history and at-most-once key.
- **PublishedPost**: the record of an actual publication — outcome, external
  reference, dry-run flag, error detail.
- **ApprovalRequest**: a pending decision on content — requester, approver, decision,
  comment, timestamp.
- **Notification**: an in-app/email message with read state.
- **AuditEvent**: the immutable record of a change.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new operator goes from a clean clone to a signed-in, seeded command
  center on a single workstation in under 15 minutes, with no manual step beyond
  providing configuration values.
- **SC-002**: An operator takes a piece of content from creation to scheduled in under
  5 minutes, including one approval round.
- **SC-003**: 99% of scheduled posts are dispatched within 15 seconds of their
  scheduled time; none are lost or duplicated across a forced restart.
- **SC-004**: 100% of state changes are attributable — every one is present in the
  audit trail with actor, time and before/after values.
- **SC-005**: Zero unapproved items reach publication across the entire automated test
  suite and a manual end-to-end run.
- **SC-006**: The dashboard reflects its underlying records exactly and loads within
  1 second on a seeded dataset.
- **SC-007**: 100% of authorization tests pass — every role is refused every operation
  it is not entitled to.
- **SC-008**: 100% of domain data survives a service restart, an upgrade and a full redeployment from the same configuration.
- **SC-009**: 90% of the first-time evaluation tasks in the acceptance criteria are
  completed by a tester without consulting documentation beyond the README.

## Assumptions

- Multi-workspace schema from day one, with each user holding one active workspace at
  a time; per-workspace isolation is proven by tests.
- Authentication is email and password with server-side sessions; no single sign-on in
  Phase 1.
- Email is delivered through a local SMTP catch-all in development and a pluggable
  provider in production.
- `viewer` may read everything the workspace shares with them, including drafts, but
  may not create, edit, approve or schedule.
- Platform publishing is exercised only through the dry-run adapter; no third-party API
  is called in Phase 1.
- Media is limited to common image and video formats with a configurable size ceiling.
- Scheduling times are stored and compared as instants with the workspace timezone used
  for display.
- The seed script's data is clearly marked as demo content.

## Deferred Capability Declaration *(Social OS preset)*

| Later phase | Capability deliberately not implemented | Extension point this spec leaves behind |
|---|---|---|
| Phase 2 | Real publishing to X / LinkedIn / Facebook / Instagram; OAuth token exchange; reading comments and analytics | Adapter interface with a capability matrix, dry-run adapter, encrypted token columns, `PublishedPost`/`Engagement`/`AnalyticsSnapshot` records |
| Phase 3 | AI drafting, rewriting, platform adaptation, classification | The `AI_REVIEWED` state traversed without a model, plus the model-provider interface and `agent_runs`/`agent_tasks` tables |
| Phase 4 | Strategist, Researcher, Content, Engagement and Analytics agents with an orchestrator | Role/permission model, agent run tables, event hooks, feedback path reserved |
| Phase 5 | Automated workflows, scheduled research, report generation | Domain events emitted by the API and a cron-shaped durable job runner with no scheduled producers |
| Phase 6 | Comment classification, sentiment/intent analysis, automated replies | `Engagement` entity, policy defaults for risky categories, human-only routing rules |
| Phase 7 | Performance learning and memory feedback into strategy | `AnalyticsSnapshot` and memory tables, event hooks, scoring fields left empty |
| Later | Media generation (image/video) | `MediaAsset` provenance field distinguishing uploaded from generated |

## Constitution Compliance Record *(Social OS preset)*

| Principle | How this spec complies | Where verified (task/test/AC id) |
|---|---|---|
| P1 Self-hosted / P2 CPU AI | Runs entirely from a clean clone on one workstation; no cloud dependency; AI steps are traversed without a model | AC-1 (clean install), SC-001 |
| P3 Adapters / P4 Official APIs | FR-013: platform access only through the adapter and capability matrix; dry-run only in Phase 1 | US-003 scenarios 1–3, US-010 scenario 2 |
| P5 FastAPI owns logic / P6 LLMProvider | All rules here live in the API/service layer; no automation tool required to run Phase 1; model interface left unwired | Plan Constitution Check gate P5 |
| P7 Least privilege / P8 Approval default | FR-003 role enforcement, FR-010 approval-by-default with separation of duties | US-002, US-009 scenario 3, SC-007, SC-005 |
| P9 Secrets / P10 State machine + audit | FR-019 keeps credentials out of the tree with tokens encrypted at rest; FR-009 and FR-015 enforce the lifecycle and audit trail | US-008 scenarios 1–2, US-013, SC-004 |
| P11 Contract-first / P12 Tests / P13 SDD / P14 Supported≠Available | This spec precedes code; every FR maps to a contract endpoint; the Phase Gate marks each capability `available` or `supported-only`; the checklist carries the evidence | `checklists/requirements.md`, `docs/API-CONTRACTS.md`, tasks closing items |
