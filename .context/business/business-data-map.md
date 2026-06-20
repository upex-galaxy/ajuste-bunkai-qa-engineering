# Business Data Map: Bunkai (QA Context)

> Generated: 2026-06-18
> Sources: target `architecture-specs.md` (ERD), `business-data-map.md`, `functional-specs.md`, `domain-glossary.md` (this repo).
> Scope: Bunkai MVP — Cloud edition.

---

## Executive Summary

**What does this system do?** Bunkai is a Test Management System. QA engineers create atomic, reusable ATCs (Atomic Test Components) anchored to User Stories + Acceptance Criteria, chain them into Tests, execute Runs manually (or via API in Phase 2), and file Bugs natively — all within a hierarchical Workspace → Project → Module structure.

**Main Actors**

| Actor | Description | Key Flows |
|---|---|---|
| QA Engineer (Elena) | Creates ATCs, Tests, Runs, Bugs | Create ATC → Chain Test → Execute Run → File Bug |
| QA Lead (Mateo) | Configures Workspace, watches dashboards | Manage Modules → View Reports → Filter Runs |
| Developer (Sara) | Checks coverage for her features | Search ATCs → View Test detail → Link Bugs |
| AI Agent (Karim) | Drives Runs via REST API | Auth → Fetch Test → Execute Run → Report Results |
| Viewer (stakeholder) | Consumes dashboards read-only | View heatmaps, coverage trends |

---

## Entity Map

### Core Entities

```
Workspace (tenant root)
  └── Project (app under test)
        ├── Module (test area — tree, depth ≤ 6)
        │     ├── User Story (feature)
        │     │     └── Acceptance Criterion (testable behavior)
        │     ├── ATC (atomic test component — anchored to US+AC)
        │     │     ├── ATC Step (ordered step)
        │     │     └── ATC Assertion (expected outcome)
        │     └── Bug (defect — anchored to Module)
        ├── Environment (target URL)
        └── Integration (Jira sync)

Test (ordered ATC chain)
  └── Test Step (ATC invocation — ordered)

Run (Test execution)
  ├── Run ATC (snapshot of ATC at run time)
  │     └── Run Step (step result: pass/fail/block)
  └── Bug (optionally linked)

User
  ├── Workspace Member (RBAC role)
  └── Access Token (PAT for API auth)
```

### Entity Details

| Entity | Key Attributes | Stateful? | Primary API |
|---|---|---|---|
| Workspace | id, name, slug | No | POST /workspaces |
| Project | id, workspace_id, name, slug | No | POST /projects |
| Module | id, project_id, parent_module_id, name, path, archived_at | Yes (archived) | POST /modules |
| UserStory | id, module_id, title, description, external_id | No | POST /user-stories |
| AcceptanceCriterion | id, user_story_id, title, position | No | POST /acceptance-criteria |
| ATC | id, module_id, title, layer, tags | Yes (via module archive) | POST /atcs |
| Test | id, title, tags | No | POST /tests |
| Run | id, test_id, status, environment_id, executor | Yes (5 states) | POST /runs |
| RunStep | id, run_atc_id, atc_step_id, status, evidence | Yes (3 states) | POST /runs/{id}/steps/{step_id}/result |
| Bug | id, module_id, atc_id, run_id, severity, status | Yes (5 states) | POST /bugs |

---

## State Machines (QA Critical)

### Run State Machine

```
[*] → created → running → passed → [*]
                    ↓
              ┌→ failed → [*]
              └→ aborted → [*]
```

**Transitions to test**:
- `created → running`: first step result posted
- `running → passed`: all ATCs pass
- `running → failed`: any step fails
- `running → aborted`: abort endpoint called
- `[idempotency]` same `idempotency_key` → same `run_id`
- `[edge]` double-finish attempt → idempotent (no error)

### Bug State Machine

```
[*] → open → in_progress → resolved → closed → [*]
                    ↓              ↑
                rejected → [*]
                   
open → reopened → [*] (reopen from closed)
```

**Transitions to test**:
- Filed from `run` step result (passes Module + ATC + Run context)
- Filed standalone via API (requires `module_id`)
- Severity P1–P4 enforced (required field)
- Optional Jira sync (bidirectional)

### Module Soft-Delete Cascade

```
Module → archived_at set
  → child Modules → archived_at set
  → User Stories → archived_at (if module-level)
  → ATCs → archived_at
  → Tests → archived_at (if anchored)
  → Bugs → archived_at
```

**Invariant**: Archived entities are excluded from dashboards and searches. They are NOT hard-deleted.

---

## Key Flows

### Flow 1: Create ATC → Chain into Test → Execute Run → File Bug

```
Elena creates ATC-001 (anchored to US-BK-12 AC-03)
  → Chains [ATC-001, ATC-002, ATC-003] into TEST-008
  → Starts RUN-045 on staging
  → Steps through ATCs (pass/fail/block per step)
  → ATC-002 step 3 fails → files BUG-014
  → Aborts RUN-045
  → Dashboard shows Payment module +1 defect
```

### Flow 2: AI Agent Nightly Regression

```
Agent (Karim) authenticates with Bearer token
  → GET /tests/TEST-008?expand=atcs.steps
  → POST /runs { test_id, idempotency_key }
  → For each ATC step: POST /runs/{id}/steps/{step_id}/result
  → On failure: POST /bugs
  → POST /runs/{id}/finish
  → Both Run + Bug indistinguishable from manual flow
```

---

## Integration Points

| Integration | Direction | Protocol | MVP Status |
|---|---|---|---|
| Jira | Bidirectional | REST API + webhook | Optional, Phase 1 import only |
| Supabase Auth | AuthN | SDK | ✅ |
| Cloudflare R2 | Storage | S3-compatible signed URLs | ✅ |
| Sentry | Error monitoring | SDK | ✅ |
| PostHog | Analytics | SDK | ✅ |

---

## System Triggers

| Trigger | Action | Entity |
|---|---|---|
| First step result | Run → running | Run |
| All steps passed | Run → passed | Run |
| Any step failed | Run → failed | Run |
| User aborts | Run → aborted | Run |
| Module deleted | cascade `archived_at` | Module + descendants |
| Bug filed | (optional) Jira sync | Bug |

---

## Discovery Gaps

- [ ] Live Supabase schema not queried (need `[DB_TOOL]` or migrate files)
- [ ] Actual Jira workflows for Bug sync not mapped
- [ ] Realtime propagation latency not measured
- [ ] Agentic execution protocol not implemented (Phase 2)
- [ ] CI/CD adapter import format not defined (Phase 2)
