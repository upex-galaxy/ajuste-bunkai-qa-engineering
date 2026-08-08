# Bunkai — Business Data Map

```
  ╔══════════════════════════════════════════════════════════════════╗
  ║                    BUNKAI (分解) — TMS                          ║
  ║       Test Management System for QA Engineers                  ║
  ║     Multi-tenant · Jira-sync · ATC-first · Supabase-backed     ║
  ╚══════════════════════════════════════════════════════════════════╝
```

## Executive Summary

Bunkai is a **Test Management System (TMS)** purpose-built for QA engineers. Teams organize testing work under **workspaces** (multi-tenant), create **projects** (the system-under-test), define a **module tree** (up to 6 levels deep), author **User Stories** with **Acceptance Criteria**, and write **Acceptance Test Cases (ATCs)** — the core entity — with ordered steps, assertions, and AC bindings.

Actors and their relationship:

```
  QA Engineer ──► Workspace ──► Project ──► Module Tree
       │                                            │
       │                                            +──► User Stories ──► ACs
       │                                            │
       │                                            +──► ATCs (Steps · Assertions · AC bindings)
       │
       +──► Jira (import stories + ACs via JQL)
       +──► PAT (CLI/Agent auth, scoped)
```

Value proposition: one place to author, organize, track, and report on test cases — with Jira bi-directional sync, RBAC workspaces, and a developer-focused dark-mode UI.

---

## Entity Map

```
auth.users
    │  (owner_user_id)
    ├──< workspaces
    │      ├──< workspace_members (RBAC: viewer|member|admin|owner)
    │      ├──< projects
    │      │      ├──< modules  (self-referential tree, max depth 6, materialized path)
    │      │      │      ├──< user_stories (draft|ready_to_test)
    │      │      │      │      └──< acceptance_criteria (positioned, soft-delete)
    │      │      │      └──< atcs  (CORE ENTITY)
    │      │      │              ├──< atc_steps (content · input_data · expected)
    │      │      │              ├──< atc_assertions (content)
    │      │      │              └──< atc_acceptance_criteria (M:N join)
    │      │      └──< import_jobs (async Jira import, one active per project)
    │      ├──< workspace_invites (email, 7d expiry, accepted|revoked|expired)
    │      └──< activity_log (audit trail)
    │
    ├──< access_tokens (PAT: bk_pat_*, scopes, browser-only issuance)
    ├──< magic_link_tokens (OTP audit trail)
    ├──< idempotency_keys (24h POST replay protection)
    ├──< feature_flags (global|workspace scoped)
    └──< user_view_state (UI persistence)
```

| Entity | Business Role | Why it exists |
|--------|---------------|---------------|
| **Workspace** | Multi-tenant boundary | Isolate QA teams. Owner manages members, plan (community → cloud → enterprise). |
| **WorkspaceMember** | RBAC access | Each user has a role per workspace. Invite flow: viewer (read-only) → member (read-write) → admin (manage) → owner (delete). |
| **Project** | System-under-test | The application your team tests. Each workspace can have many projects. |
| **Module** | Hierarchical organization | Tree of features/modules (max 6 deep). Materialized path for fast queries. Supports rename, move (with cycle/depth guard), soft-delete subtree. |
| **UserStory** | Business intent | A feature or requirement tied to a module. Status gates: `draft` → `ready_to_test` requires ≥1 AC. Soft-deletable. Optional Jira `external_id`. |
| **AcceptanceCriterion** | Testable condition | The "what" of testing. Belongs to a story, ordered by `position`. Last-AC-archived auto-reverts story to `draft`. Soft-deletable (parking-lot technique for reordering). |
| **ATC** | Formal test case | THE core entity. A complete test: steps + assertions + AC bindings. Has slug, layer (UI/API/Unit), version (optimistic lock), status lifecycle (unrun → pass/fail/blocked/skipped/running), tags, full-text search. Soft-deletable. Must anchor to ≥1 AC. |
| **AtcStep** | Test step | Ordered: content, input_data, expected result. |
| **AtcAssertion** | Test assertion | Ordered: what to assert. |
| **AtcAcceptanceCriterion** | Traceability link | M:N join ensuring every ATC is traceable to ≥1 AC within the same User Story. |
| **AccessToken** | PAT for CLI/Agent | `bk_pat_*` prefix. Scopes: atc:read, atc:write, run:execute, workspace:admin. Browser-session-only issuance (PAT cannot mint PAT). |
| **ImportJob** | Jira import | Async JQL-based import. One active per project (partial unique index). Status: queued → running → completed/failed. |
| **ActivityLog** | Audit trail | Immutable log of entity mutations (atc.created, story.updated, etc.). |
| **IdempotencyKey** | POST safety | 24h TTL, (user, endpoint, key) scoped. Prevents duplicate ATC/AC creation on retry. |

---

## Business Flows

### Flow 1: Workspace Onboarding (Sign-up + First Workspace)

```
User ──► /auth/signup ──► POST /api/v1/auth/signup ──► auth.users created
                                                           │
                                                           ▼
User ──► POST /api/v1/workspaces ──► bunkai_bootstrap_workspace()
                                                           │
                                      ┌────────────────────┤
                                      ▼                    ▼
                               workspace created      owner membership inserted
                                                           │
                                      ┌────────────────────┤
                                      ▼                    ▼
                               cookie session set     PAT minted (response only)
```

**Business rules:**
- Signup auto-confirms (no email verification)
- Bootstrap creates workspace + owner membership atomically (SECURITY DEFINER)
- Workspace plan defaults to `community`
- PAT returned in signup response — shown once only

### Flow 2: Jira Import

```
User ──► POST /api/v1/imports { jql, projectId }
           │
           ▼
    Insert import_job (queued)
           │
           ▼
    Vercel after() background slot
           │
           ▼
    import_run() ──► Jira API (paginated, up to 1000 pages)
                        │
                        ▼
           For each issue: upsert user_story + ACs
                        │
           ┌────────────┴────────────────┐
           │                              │
      Insert/Update                 Log errors
      stories + ACs                      │
           │                              ▼
           ▼                      status = 'failed'
    status = 'completed'
```

**Business rules:**
- One active import per project (partial unique index enforces)
- Race-free via atomic status claim (queued → running, WHERE current_timestamp)
- Jira auth token from DB, not env
- Errors collected per-item, available via GET /imports/{id}

### Flow 3: ATC Authoring (Create)

```
User ──► POST /api/v1/atcs { moduleId, storyId, steps[], assertions[], acIds[] }
           │
           ▼
    Principal check (requires `atc:write` scope)
           │
           ▼
    bunkai_create_atc() — single RPC transaction
           │
      ┌────┴─────────────────────────────────┐
      │     Validate acIds belong to story       │
      │     Generate slug (module-path/atc-*)    │
      │     Increment version (starts at 1)      │
      │     Insert ATC row                       │
      │     Insert steps (ordered)               │
      │     Insert assertions (ordered)          │
      │     Insert AC bindings                   │
      │     Insert activity_log event            │
      │     Return full ATC with children        │
      └─────────────────────────────────────────┘
```

**Business rules:**
- ATC must anchor to ≥1 AC belonging to the parent User Story
- ATC's module must be the story's module or a descendant thereof
- Steps and assertions are full-replaced on update (PATCH /atcs/{id})
- Optimistic locking via `X-If-Match` header + `version` column
- PAT issuance requires `atc:write` scope

### Flow 4: Module Tree Management

```
User ──► POST /modules          (create)
User ──► PATCH /modules/{id}   (rename → path rebuild cascade)
User ──► POST /modules/{id}/move (re-parent → cycle+ depth check)
User ──► DELETE /modules/{id}   (soft-delete → recursive CTE archive)
```

**Business rules:**
- Max depth: 6 levels (enforced in RPC)
- Cycle detection: path-prefix check prevents ancestor-as-descendant moves
- Rename: rebuilds materialized `path` for module + ALL descendants in one UPDATE
- Delete: recursive CTE archives module + stories + ACs + ATCs in the subtree
- Uniqueness: `(project_id, path)` unique, case-insensitive slug for siblings

### Flow 5: Acceptance Criteria Management

```
User ──► POST   /user-stories/{id}/acceptance-criteria (create + auto-position)
User ──► PATCH  /acceptance-criteria/{id}               (move position)
User ──► DELETE /acceptance-criteria/{id}               (archive)
           │
           ▼
    If last AC archived → auto-revert story to `draft`
```

**Business rules:**
- Position rebalance uses negative-parking (collision-free concurrent inserts)
- Last-AC-removal causes story status auto-revert (draft gate)
- Story transition `draft → ready_to_test` requires ≥1 active AC (FOR UPDATE lock)

### Flow 6: Authentication & Authorization

```
Browser:  POST /auth/magic-link → email OTP → callback → cookie session
           or POST /auth/signin  → email+password → cookie session + PAT

CLI:      Bearer bk_pat_XXXXX → resolveIdentity() → Principal
           │
           ▼
    Supabase RLS client (scoped to caller's workspace memberships)
           │
           ▼
    Postgres enforces: can caller read/write this row?
```

**Dual auth parity:**
- Both auth modes collapse to a single `Principal` shape
- Both use the same RLS-scoped Supabase client
- Authorization is Postgres-enforced, not TypeScript-level
- PAT issuance is browser-session-only (PAT cannot mint PAT)

---

## State Machines

### ATC Status (`atcs.status`)

```
           ┌─────────┐
           │  unrun  │────────────┐
           └────┬────┘            │
                │                 │
                ▼                 │
           ┌─────────┐           │
           │ running │           │
           └────┬────┘           │
                │                 │
       ┌────────┼────────┐       │
       ▼        ▼        ▼       │
   ┌──────┐ ┌──────┐ ┌──────┐   │
   │ pass │ │ fail │ │blocked│   │
   └──────┘ └──────┘ └──────┘   │
       │        │        │       │
       └────────┴────────┴──┬────┘
                            │
                            ▼
                    Any state → any (full cycle allowed)
```

| From | To | Event | Effects |
|------|----|-------|---------|
| unrun | running | Test execution started | Locks ATC row |
| running | pass | All assertions met | Updates status, logs activity |
| running | fail | Assertion(s) failed | Updates status, logs activity |
| running | blocked | Cannot proceed | Dependency / env issue |
| running | skipped | Intentionally skipped | — |
| Any | unrun | Reset | Returns to initial state |

**Business consequence:** ATC status drives test execution reporting. An ATC stuck in `running` (e.g., crashed runner) blocks accurate pass/fail metrics.

### User Story Status (`user_stories.status`)

```
  ┌───────┐     ≥1 AC active      ┌──────────────┐
  │ draft │ ──────────────────►   │ready_to_test │
  └───┬───┘                      └──────┬───────┘
      │                                  │
      └── Last AC archived ◄─────────────┘
```

| From | To | Event | Effects |
|------|----|-------|---------|
| draft | ready_to_test | PATCH story status | Rejected if 0 active ACs (FOR UPDATE lock) |
| ready_to_test | draft | Last AC archived | Auto-revert via DB function |

**Business consequence:** A story without ACs cannot be marked ready. This enforces BDD-quality: you must define what "done" means before testing begins.

### Member Role & Status

```
Role hierarchy:  viewer (read) < member (write) < admin (manage) < owner (full)

Status:  invited ──► active ◄── suspended
                │
                └── expires (7d) ──► (no active membership)

Rule: No demotion on invite accept (≥ existing role → reject)
```

**Business consequence:** Role escalation bugs could leak write access to viewers. Status bypass could allow suspended members to access data.

### Invite Status (derived)

```
pending ──► accepted (user accepts with matching email)
     │
     ├──► revoked (admin revokes)
     └──► expired (7 days TTL)
```

### Import Job Status

```
queued ──► running ──► completed
                  │
                  └──► failed
```

---

## Automatic Processes

### DB Triggers

| Trigger | Fires On | Action | Why it exists |
|---------|----------|--------|---------------|
| `atcs_set_updated_at` | UPDATE on `atcs` | Sets `updated_at = now()` | Every ATC mutation should update the timestamp |
| `atcs_refresh_tsv` | INSERT/UPDATE of title/tags on `atcs` | Rebuilds `tsvector` for full-text search | Keeps ATC search current without manual refresh |
| AC auto-revert | Archive of last AC | Reverts story to `draft` | Enforces the ≥1 AC invariant automatically |

### Cron Jobs

None detected in the codebase.

### Vercel `after()` Background Processes

| Process | Trigger | What it does | Why it exists |
|---------|---------|--------------|---------------|
| Jira Import Runner | POST `/api/v1/imports` + DB insert | Pages Jira API, upserts stories + ACs | Long-running import must not block HTTP response |
| (Future) Invite email | POST workspace invite | Would send via Resend | MVP logs to console only |

### Webhooks

None detected in the MVP.

---

## External Integrations

### Supabase (PostgreSQL + Auth + RLS)

```
Bunkai App ──► Supabase Client ──► Postgres
                  │                      │
                  │               + RLS policies
                  │               + DB functions
                  │
           Auth: magic link,
           email+password
```

- **Dependent flows:** ALL of them. Every API call resolves to a Supabase query.
- **Failure behavior:** App is non-functional without DB. Auth stops, queries fail, RLS denies.
- **Critical timeout:** Default Supabase client timeout. Retry on transient failures.
- **Known quirk:** RLS policies are the authorization layer; if a policy typo grants too much, TS code has no second gate.

### Jira

```
Bunkai ──► Jira REST API (paginated JQL queries)
              │
              ▼
       Import: issues → user_stories + ACs
```

- **Dependent flows:** Import only (Flow 2). Not on critical read/write path.
- **Failure behavior:** Import job fails with error collection. Existing data unaffected.
- **Acceptable degradation:** Import unavailable → manual ATC creation works fine.
- **Known quirks:** Pagination up to 1000 pages. JQL syntax varies. ADF markdown conversion is lossy.

### Resend (Email — configured, not yet wired)

- **Dependent flows:** None currently (invite emails log to console).
- **Acceptable degradation:** N/A — not production-critical yet.

---

## Discovery Gaps

- **Live DB schema not inspected** — this map was built from migration files and source code, not a live `information_schema` query. Edge cases in constraints (deferrable, partial indexes beyond known ones) may be undocumented.
- **Live API not exercised** — endpoint inventory derived from route files. Actual request/response schemas may differ or include undocumented fields.
- **Jira import error modes** — The import runner collects errors, but error types (rate limit, auth expiry, schema change) are not exhaustively documented.
- **RLS policies not read** — Authorization rules derived from code, not from `pg_policies`. TS code may not match actual Postgres-enforced rules.
- **Feature flag behavior** — `feature_flags` table exists but MVP usage in code is untraced.
- **No cron jobs confirmed** — Supabase scheduled functions or Vercel CRON may exist beyond codebase analysis.
