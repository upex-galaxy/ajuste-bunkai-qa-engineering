# Functional Specifications — Bunkai (QA Context)

> Source: target `.context/SRS/functional-specs.md`. 40 FRs across 7 MVP epics.
> This is a QA-focused index. Full detail in target's `functional-specs.md`.

---

## Epic Index

| Epic | FRs | Domain |
|---|---|---|
| EPIC-BK-001 | BK-001–004 | Tenancy & Identity |
| EPIC-BK-002 | BK-005–006 | Project & Module Hierarchy |
| EPIC-BK-003 | BK-007–010 | User Stories & Acceptance Criteria |
| EPIC-BK-004 | BK-011–016 | ATC Library |
| EPIC-BK-005 | BK-017–020 | Tests as ATC Chains |
| EPIC-BK-006 | BK-021–026 | Manual Execution + Runs |
| EPIC-BK-007 | BK-027–030 | Bugs |
| EPIC-BK-008 | BK-031–035 | Views & Search |
| EPIC-BK-009 | BK-036–040 | API + CLI Foundation |

## Key Business Rules (must test)

| Rule | FR | Invariant | Test |
|---|---|---|---|
| ATC must be anchored to ≥1 US + AC | BK-011 | `atc_acceptance_criteria` must have ≥1 row | Attempt to create ATC without US/AC → 400 |
| Module depth ≤ 6 | BK-006 | Recursive CTE check | Create depth-7 module → 409 |
| Soft-delete cascade | BK-006, BK-039 | `archived_at` propagates to descendants | Delete module → verify child entities archived |
| Role hierarchy | BK-003 | Cannot invite role higher than caller | member tries to invite admin → 403 |
| Run idempotency | BK-021 | Same key → same `run_id` | POST twice with same key → 201 then 200 |
| Bug must link to Module | BK-027 | `module_id` required | POST bug without module → 400 |

## API Endpoint Map (MVP)

| Method | Path | FR | Notes |
|---|---|---|---|
| POST | `/api/v1/auth/signup` | BK-001 | Email + OAuth |
| POST | `/api/v1/workspaces` | BK-002 | Create workspace |
| POST | `/api/v1/workspaces/{id}/invites` | BK-003 | Invite teammate |
| GET | `/api/v1/workspaces` | BK-004 | List; switch active |
| POST | `/api/v1/projects` | BK-005 | Create project |
| POST | `/api/v1/modules` | BK-006 | CRUD + nesting |
| POST | `/api/v1/user-stories` | BK-007 | CRUD |
| POST | `/api/v1/user-stories/{id}/acceptance-criteria` | BK-008 | AC CRUD |
| POST | `/api/v1/atcs` | BK-011 | ATC CRUD (anchored) |
| GET | `/api/v1/atcs?search=` | BK-015 | Text search |
| POST | `/api/v1/tests` | BK-017 | Test CRUD (ATC chain) |
| POST | `/api/v1/runs` | BK-021 | Start run (idempotent) |
| POST | `/api/v1/runs/{id}/steps/{step_id}/result` | BK-023 | Report step result |
| POST | `/api/v1/runs/{id}/finish` | BK-025 | Finish run |
| POST | `/api/v1/bugs` | BK-027 | File bug |
| GET | `/api/v1/bugs?module=` | BK-030 | Filter by module |
| GET | `/api/v1/me` | BK-036 | Current user |
| POST | `/api/v1/access-tokens` | BK-038 | Create PAT |

## QA Testing Priorities

1. **Data-model invariants** — ATC anchoring, Module cascade, Run idempotency (highest risk)
2. **State machines** — Run lifecycle (created→running→passed/failed/aborted), Bug lifecycle
3. **API contract** — every endpoint returns `{success, data, error}` envelope, error codes are machine-readable
4. **RBAC enforcement** — viewer/member/admin/owner boundaries at API level
5. **Jira sync** — bidirectional, conflict resolution, failure handling
6. **Realtime** — UI updates via Supabase Realtime during Run execution
