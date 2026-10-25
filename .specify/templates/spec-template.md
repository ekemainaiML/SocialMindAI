# Feature Specification: [FEATURE NAME]

**Feature Branch**: `[###-feature-name]`

**Created**: [DATE]

**Status**: Draft

**Input**: User description: "$ARGUMENTS"

**Constitution**: `.specify/memory/constitution.md` (P1–P14). Every section below must
comply with it; violations are recorded in the compliance table at the bottom, not
silently accepted.

## Phase Gate *(Social OS preset — mandatory)*

<!-- The project builds in seven phases. A feature belongs to exactly one phase and must
     not implement later-phase capability; it may only define the extension point. -->

- **Phase**: [1 command center | 2 platform integrations | 3 local AI | 4 agents | 5 automation | 6 autonomous engagement | 7 learning]
- **Constitution check**: P3 (adapter-only platform logic) · P5 (FastAPI owns the logic) ·
  P8 (approval default) · P9 (no secrets in tree) · P11 (contract updated) · P13 (loop
  followed) — state pass/fail per principle.
- **Supported vs Available (P14)**: list each capability this feature touches and mark it
  `available` (live, permitted API path) or `supported-only` (extension point, stub,
  planned). Cite the source of the permission assumption (e.g. LinkedIn member vs
  organization scopes).

## User Scenarios & Testing *(mandatory)*

<!--
  IMPORTANT: User stories should be PRIORITIZED as user journeys ordered by importance.
  Each user story/journey must be INDEPENDENTLY TESTABLE - meaning if you implement just ONE of them,
  you should still have a viable MVP (Minimum Viable Product) that delivers value.

  Assign priorities (P1, P2, P3, etc.) to each story, where P1 is the most critical.
  Think of each story as a standalone slice of functionality that can be:
  - Developed independently
  - Tested independently
  - Deployed independently
  - Demonstrated to users independently
-->

### User Story 1 - [Brief Title] (Priority: P1)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently - e.g., "Can be fully tested by [specific action] and delivers [specific value]"]

**Acceptance Scenarios**:

1. **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 2 - [Brief Title] (Priority: P2)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently]

**Acceptance Scenarios**:

1. **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 3 - [Brief Title] (Priority: P3)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [Describe how this can be tested independently]

**Acceptance Scenarios**:

1. **Given** [initial state], **When** [action], **Then** [expected outcome]

---

[Add more user stories as needed, each with an assigned priority]

### Edge Cases

<!--
  ACTION REQUIRED: The content in this section represents placeholders.
  Fill them out with the right edge cases.
-->

- What happens when [boundary condition]?
- How does system handle [error scenario]?

## Requirements *(mandatory)*

<!--
  ACTION REQUIRED: The content in this section represents placeholders.
  Fill them out with the right functional requirements.
-->

### Functional Requirements

- **FR-001**: System MUST [specific capability, e.g., "allow users to create accounts"]
- **FR-002**: System MUST [specific capability, e.g., "validate email addresses"]
- **FR-003**: Users MUST be able to [key interaction, e.g., "reset their password"]
- **FR-004**: System MUST [data requirement, e.g., "persist user preferences"]
- **FR-005**: System MUST [behavior, e.g., "log all security events"]

*Example of marking unclear requirements:*

- **FR-006**: System MUST authenticate users via [NEEDS CLARIFICATION: auth method not specified - email/password, SSO, OAuth?]
- **FR-007**: System MUST retain user data for [NEEDS CLARIFICATION: retention period not specified]

### Key Entities *(include if feature involves data)*

- **[Entity 1]**: [What it represents, key attributes without implementation]
- **[Entity 2]**: [What it represents, relationships to other entities]

## Success Criteria *(mandatory)*

<!--
  ACTION REQUIRED: Define measurable success criteria.
  These must be technology-agnostic and measurable.
-->

### Measurable Outcomes

- **SC-001**: [Measurable metric, e.g., "Users can complete account creation in under 2 minutes"]
- **SC-002**: [Measurable metric, e.g., "System handles 1000 concurrent users without degradation"]
- **SC-003**: [User satisfaction metric, e.g., "90% of users successfully complete primary task on first attempt"]
- **SC-004**: [Business metric, e.g., "Reduce support tickets related to [X] by 50%"]

## Assumptions

<!--
  ACTION REQUIRED: The content in this section represents placeholders.
  Fill them out with the right assumptions based on reasonable defaults
  chosen when the feature description did not specify certain details.
-->

- [Assumption about target users, e.g., "Users have stable internet connectivity"]
- [Assumption about scope boundaries, e.g., "Mobile support is out of scope for v1"]
- [Assumption about data/environment, e.g., "Existing authentication system will be reused"]
- [Dependency on existing system/service, e.g., "Requires access to the existing user profile API"]

## Deferred Capability Declaration *(Social OS preset)*

<!-- Every feature must say explicitly what it is NOT building, and which extension
     point it leaves for the later phase. Prevents Phase N from growing Phase N+1. -->

| Later phase | Capability deliberately not implemented | Extension point this spec leaves behind |
|---|---|---|
| [e.g. Phase 2] | [e.g. real LinkedIn publishing] | [e.g. `SocialPlatformAdapter.dispatch()` + `DryRunAdapter`] |

## Constitution Compliance Record *(Social OS preset)*

| Principle | How this spec complies | Where verified (task/test/AC id) |
|---|---|---|
| P1 Self-hosted / P2 CPU AI | | |
| P3 Adapters / P4 Official APIs | | |
| P5 FastAPI owns logic / P6 LLMProvider | | |
| P7 Least privilege / P8 Approval default | | |
| P9 Secrets / P10 State machine + audit | | |
| P11 Contract-first / P12 Tests / P13 SDD / P14 Supported≠Available | | |
