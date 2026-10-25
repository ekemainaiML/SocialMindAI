# AGENTS.md — Working on Social OS

**This repository is developed with Spec-Driven Development (SDD).** No production code
is written before its spec, plan and task list exist and its checklist is evidenced.

Source of truth for the product vision: [Social Media OS Development.docx](Social%20Media%20OS%20Development.docx).
Brand asset: [logo.jpg](logo.jpg).

---

## 1. The loop

```
constitution  .specify/memory/constitution.md      (once, then amendments only)
     ↓
specify       specs/NNN-feature/spec.md            what & why, user stories, ACs
     ↓
clarify       unresolved points resolved into decisions (inline in spec.md)
     ↓
plan          specs/NNN-feature/plan.md            research, data model, contracts,
               + research.md, data-model.md,        technical context, constitution gate
               quickstart.md, contracts/
     ↓
checklist     specs/NNN-feature/checklists/requirements.md   requirements quality
     ↓
tasks         specs/NNN-feature/tasks.md           independently implementable tasks,
                                                  grouped by user story
     ↓
analyze       cross-artifact consistency report (optional but recommended)
     ↓
implement     code + tests, one task at a time
     ↓
converge      verify against spec, record evidence, append remaining work, repeat
```

Each step is a **human checkpoint**: review the artifact before the next step runs.

## 2. How to run the steps here

Spec Kit installed its process as agent **skills** in
[`.claude/skills/`](.claude/skills/) — they are chat skills, not shell commands. To run a
step, read the matching `SKILL.md` and follow it exactly:

| Step | Skill |
|---|---|
| constitution | `.claude/skills/speckit-constitution/SKILL.md` |
| specify | `.claude/skills/speckit-specify/SKILL.md` |
| clarify | `.claude/skills/speckit-clarify/SKILL.md` |
| plan | `.claude/skills/speckit-plan/SKILL.md` |
| checklist | `.claude/skills/speckit-checklist/SKILL.md` |
| tasks | `.claude/skills/speckit-tasks/SKILL.md` |
| analyze | `.claude/skills/speckit-analyze/SKILL.md` |
| implement | `.claude/skills/speckit-implement/SKILL.md` |
| converge | `.claude/skills/speckit-converge/SKILL.md` |

The skills rely on helper scripts in [`.specify/scripts/bash/`](.specify/scripts/bash/)
(`create-new-feature.sh`, `check-prerequisites.sh`, `setup-plan.sh`, `setup-tasks.sh`,
`resolve-template.sh`). Run them from the repository root; they create the feature
directory and branch (`specs/NNN-feature/`, branch `NNN-feature`).

Templates these skills fill in (already customized for this project):
`.specify/templates/{spec,plan,tasks,checklist,constitution}-template.md`.

## 3. Where documents live

| Location | Content | Maintained by |
|---|---|---|
| `.specify/memory/constitution.md` | **Canonical constitution (P1–P14)** — edit here first | `/speckit-constitution` |
| `docs/CONSTITUTION.md` | Human index of the constitution; mirrors the canonical file | manual mirror |
| `docs/PHASE-1-SPEC.md` | Baseline specification for Phase 1 (Social Command Center) | `/speckit-specify` lineage |
| `docs/ARCHITECTURE.md` | Repo layout, services, data model, jobs, config, local dev | `/speckit-plan` lineage |
| `docs/API-CONTRACTS.md` | REST + event + adapter contracts (contract-first, P11) | contract changes first |
| `docs/ACCEPTANCE-CRITERIA.md` | Provable ACs for Phase 1 with evidence columns | `/speckit-checklist` lineage |
| `specs/NNN-feature/` | Per-feature spec, plan, tasks, checklists, research, contracts | the loop above |

**Phase 1 feature work**: `docs/PHASE-1-SPEC.md` is authoritative until a
`specs/NNN-*` folder exists for a slice; when one exists, the feature folder governs its
own scope and links back to the baseline.

## 4. Numbering

Feature numbers follow the seven phases, e.g. `001-command-center`,
`004-oauth-linkedin`, `012-dryrun-publish`. Phases: 1 command center · 2 platform
integrations · 3 local AI · 4 agents · 5 automation · 6 autonomous engagement ·
7 learning.

## 5. Rules that are not negotiable

1. **Constitution first.** Read `.specify/memory/constitution.md` before any spec or plan.
   A plan's Constitution Check must pass all 14 gates, or record the violation in
   Complexity Tracking with justification.
2. **Supported ≠ available (P14).** Every capability claim is marked `available` or
   `supported-only`, with its permission source. Phase N does not implement Phase N+1 —
   it leaves the extension point.
3. **Contract before code (P11).** Change `docs/API-CONTRACTS.md` first, then
   implementation, then regenerate `openapi.json`; drift fails CI.
4. **Evidence before "done" (P12).** `checklist.md` holds commands and their results.
   A green build alone is not evidence. Never skip, weaken or suppress a check to pass.
5. **No secrets in the tree (P9).** Keys go in `.env` / the platform secret store;
   tokens are encrypted at rest.
6. **Fail closed.** Unknown permission, ambiguous intent, missing policy → deny and ask
   the human.

## 6. Verification

Once Phase 1 code exists, the single local gate is:

```bash
make check   # lint + typecheck + tests — identical to CI
```

Until then, the gate for documentation work is: all six `docs/` documents consistent
with each other, with the constitution, and with the source `.docx`.

## 7. Stack (Phase 1, pinned by [ARCHITECTURE.md](docs/ARCHITECTURE.md))

Next.js + TypeScript · FastAPI + Python · PostgreSQL (pgvector) · Redis · MinIO ·
Docker Compose · Caddy · Mailpit (dev) · n8n/Ollama/Prometheus profiles off by default.
