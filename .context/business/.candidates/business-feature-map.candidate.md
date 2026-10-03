# Business Feature Map — upex-bunkai-tms

> Feature-centric inventory of every capability the system offers. Complement to `business-data-map.md` (data-centric).
> **CANDIDATE** — generated in UPDATE mode. Not yet applied to `business-feature-map.md`.
> Status vocabulary: `Stable` (shipped, in production use) · `Beta` (shipped, experimental) · `Planned` (designed/partial, not usable end-to-end).
> Regenerated: 2026-09-27 · Baseline superseded: 577-line MVP-era map (baseline highest ID `FEAT-SYS-004`).

---

## 1. Inventory summary

| Category | Features | Status |
|---|---|---|
| Core (test management spine: projects, modules, stories, ATCs, tests, plans, runs, bugs, coverage) | 24 | Stable |
| Secondary (platform: auth, workspace, home, notifications, billing, export, environments, metrics, system, PAT) | 25 | Stable |
| Beta | 1 | Testing (FEAT-JIRA-001) |
| Planned | 0 | — (capability-level WIP tracked in §7 / §9) |
| **Total** | **50** | 49 Stable · 1 Beta |

- **Domains:** 22 (10 carried from baseline + **12 NEW**: Home, Test Definitions, Test Plans, Test Runs, Bugs, Milestones, Notifications, Billing, Data Export, Environments, Coverage & Traceability, Metrics)
- **API routes:** 94 route files under `app/api/` (+ `/openapi`, `/v1` root)
- **Pages:** 38 `page.tsx` (app routes)
- **Migrations:** `0001`–`0088` (gap: `0079`, `0080` missing from sequence)

---

## 2. Feature catalog (by domain)

### Domain: Authentication

#### Feature: Sign-up

| Aspect | Value |
|---|---|
| **ID** | FEAT-AUTH-001 |
| **Status** | Stable |
| **Endpoints** | `POST /v1/auth/signup`, `POST /v1/me/active-workspace` (onboarding workspace switch) |
| **UI** | `(auth)/login/page.tsx` (register mode), `(app)/onboarding/page.tsx` |
| **Users** | Anonymous visitor |
| **Dependencies** | Supabase Auth |
| **Evidence** | `app/api/v1/auth/signup/route.ts` |

**Capabilities:**
- [x] Email/password account creation
- [x] Initial workspace bootstrap
- [ ] Post-signup email confirmation UX unverified (see FEAT-AUTH-005)

#### Feature: Sign-in

| Aspect | Value |
|---|---|
| **ID** | FEAT-AUTH-002 |
| **Status** | Stable |
| **Endpoints** | `POST /v1/auth/signin` |
| **UI** | `(auth)/login/page.tsx` — KATA `LoginPage` |
| **Users** | Existing account |
| **Dependencies** | Supabase Auth |
| **Evidence** | `app/api/v1/auth/signin/route.ts`; KATA `AuthApi.authenticateSuccessfully` (PROJ-101), `loginWithInvalidCredentials` (PROJ-102) |

**Capabilities:**
- [x] Credential sign-in, session cookie
- [x] Failure path (invalid credentials) exercised by KATA ATC

#### Feature: Magic Link

| Aspect | Value |
|---|---|
| **ID** | FEAT-AUTH-003 |
| **Status** | Stable |
| **Endpoints** | `POST /v1/auth/magic-link` |
| **UI** | Login page (magic-link mode) |
| **Users** | Anonymous visitor |
| **Dependencies** | Supabase Auth + Resend (email delivery) |
| **Evidence** | `app/api/v1/auth/magic-link/route.ts` + `route.test.ts` |

**Capabilities:**
- [x] Magic-link request endpoint
- [x] Unit-tested route

#### Feature: Session introspection & workspace switching

| Aspect | Value |
|---|---|
| **ID** | FEAT-AUTH-004 |
| **Status** | Stable |
| **Endpoints** | `GET /v1/me`, `POST /v1/me/active-workspace` |
| **UI** | App shell (all `(app)` routes), workspace switcher |
| **Users** | Authenticated member |
| **Dependencies** | Supabase Auth (cookie + PAT accepted) |
| **Evidence** | `app/api/v1/me/route.ts`, `app/api/v1/me/active-workspace/route.ts` (both unit-tested) |

**Capabilities:**
- [x] Current-user profile + role resolution
- [x] Active-workspace persistence
- [x] PAT coexistence with cookie auth (`lib/api/auth-coexistence`, `lib/api/pat`)

#### Feature: Email status / confirmation / resend

| Aspect | Value |
|---|---|
| **ID** | FEAT-AUTH-005 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/auth/check-email`, `POST /v1/auth/confirm`, `POST /v1/auth/resend` |
| **UI** | Login flow (email-status screens) |
| **Users** | Anonymous visitor with unconfirmed email |
| **Dependencies** | Supabase Auth + Resend |
| **Evidence** | `app/api/v1/auth/check-email/route.ts`, `auth/confirm`, `auth/resend` (+ `route.test.ts`); migration `0034` |

**Capabilities:**
- [x] Check whether email is registered/confirmed
- [x] Confirmation endpoint
- [x] Resend confirmation email

---

### Domain: Workspace

#### Feature: Workspace CRUD

| Aspect | Value |
|---|---|
| **ID** | FEAT-WS-001 |
| **Status** | Stable |
| **Endpoints** | `GET /v1/workspaces`, `POST /v1/workspaces`, `GET /v1/workspaces/[id]`, `PATCH /v1/workspaces/[id]` |
| **UI** | `(app)/settings/workspaces/page.tsx`, workspace switcher |
| **Users** | Authenticated user (owner/admin for mutations) |
| **Dependencies** | Supabase (RLS + RPCs) |
| **Evidence** | `app/api/v1/workspaces/route.ts` (`route.test.ts`), `workspaces/[id]/route.ts` (`deletion-response.test.ts`) |

**Capabilities:**
- [x] Create workspace, list mine, read detail, rename/settings
- [ ] Delete is soft+restore — see FEAT-WS-003

#### Feature: Invites

| Aspect | Value |
|---|---|
| **ID** | FEAT-WS-002 |
| **Status** | Stable |
| **Endpoints** | `POST/GET /v1/workspaces/[id]/invites`, `POST/DELETE /v1/workspaces/[id]/invites/[inviteId]`, `POST /v1/invites/accept` |
| **UI** | `(app)/workspaces/[id]/members/page.tsx`, `invites/accept/page.tsx` |
| **Users** | Owner/admin creates; invitee accepts |
| **Roles** | `viewer \| member \| admin` only (ownership transferred, never invited) |
| **Dependencies** | Supabase RPC `bunkai_accept_invite` |
| **Evidence** | `app/api/v1/workspaces/[id]/invites/route.ts`, `invites/[inviteId]/route.ts` (re-issue = POST, revoke = DELETE); `lib/workspaces/invites.test.ts` |

**Capabilities:**
- [x] Create invite with role, list invites, re-issue (same email), revoke, accept
- [ ] **Invite email delivery NOT wired** — no Resend call in route or `lib/workspaces/invites.ts` (copy-link flow assumed) → Discovery Gap §9-2

#### Feature: Workspace deletion & restore (30-day soft delete → purge)

| Aspect | Value |
|---|---|
| **ID** | FEAT-WS-003 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `DELETE /v1/workspaces/[id]` (soft), `POST /v1/workspaces/[id]/restore` |
| **UI** | `workspaces/[id]/restore/page.tsx` |
| **Users** | Owner |
| **Dependencies** | `workspace_deletions` table, cron purge, Storage bucket cleanup, Resend (deletion email) |
| **Evidence** | `workspaces/[id]/route.ts` DELETE, `restore/route.ts`; migrations `0084`/`0084a`; `lib/workspace-deletion/email.test.ts` |

**Capabilities:**
- [x] Soft delete with `purge_deadline`, restore within window
- [x] Purge job (`row_count_digest`, zero-policy table, RPC-only)
- [x] Member notification email on deletion

#### Feature: Membership & roles

| Aspect | Value |
|---|---|
| **ID** | FEAT-WS-004 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `DELETE /v1/workspaces/[id]/membership` (leave), `GET /v1/me` (role resolution) |
| **UI** | Members page (leave action), account settings |
| **Users** | Any member (leave); owner (blocked from last-member exit) |
| **Evidence** | `workspaces/[id]/membership/route.ts` (`route.test.ts`), `lib/account/leave-workspace.test.ts`, `lib/account/me-role.test.ts` |

**Capabilities:**
- [x] Leave workspace (owner last-member exit blocked)
- [x] Role labels/resolution
- [ ] **Admin role change on existing member — no endpoint** (`membership` is DELETE-only) → Discovery Gap §9-6
- [ ] **Admin member removal — no endpoint** → Discovery Gap §9-6
- [ ] Ownership-transfer mechanism not found in routes → Discovery Gap §9-13

---

### Domain: Home Dashboard

#### Feature: Home dashboard

| Aspect | Value |
|---|---|
| **ID** | FEAT-HOME-001 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/workspaces/[id]/active-runs`, `/recent-projects`, `/open-bugs`, `/coverage`, `GET /v1/activity` |
| **UI** | `(app)/home/page.tsx` (welcome banner + 4 widgets), `(app)/activity/page.tsx` |
| **Users** | Authenticated member |
| **Evidence** | `app/(app)/home/page.tsx`; widget routes above |

**Capabilities:**
- [x] Active runs, recent projects, open bugs, coverage widgets
- [x] Activity feed

---

### Domain: Project & Module

#### Feature: Project CRUD

| Aspect | Value |
|---|---|
| **ID** | FEAT-PRJ-001 |
| **Status** | Stable |
| **Endpoints** | `POST /v1/workspaces/[id]/projects` (create), `GET /v1/workspaces/[id]` + `/recent-projects` (read), plan-limit enforced by DB trigger |
| **UI** | `(app)/projects/page.tsx`, `(app)/projects/new/page.tsx`, `(app)/projects/[projectSlug]/page.tsx` |
| **Users** | Member+ (create bounded by plan: 3 community / 50 cloud / ∞ enterprise, SQLSTATE `45700`) |
| **Evidence** | `workspaces/[id]/projects/route.ts`; migration `0048` plan-limit trigger; `lib/projects/validation.test.ts` |

**Capabilities:**
- [x] Create, read/list (via workspace detail + recent)
- [ ] **Update — no `PATCH /projects/[id]` route exists**
- [ ] **Delete — no `DELETE /projects/[id]` route exists**
- → Discovery Gap §9-4

#### Feature: Module tree

| Aspect | Value |
|---|---|
| **ID** | FEAT-MOD-001 |
| **Status** | Stable |
| **Endpoints** | `POST /v1/projects/[id]/modules`, `PATCH/DELETE /v1/modules/[id]` |
| **UI** | Project sidebar tree (module navigation on all project pages) |
| **Users** | Member+ |
| **Constraints** | Self-referential tree, max depth 6, materialized path, `archived_at` soft delete |
| **Evidence** | `projects/[id]/modules/route.ts`, `modules/[id]/route.ts`; `lib/modules/path.test.ts`, `lib/tree.test.ts` |

**Capabilities:**
- [x] Create nested modules, rename/move (PATCH), archive (DELETE)
- [x] Every artifact anchors to a leaf module

---

### Domain: User Stories

#### Feature: User story management

| Aspect | Value |
|---|---|
| **ID** | FEAT-US-001 |
| **Status** | Stable |
| **Endpoints** | `POST/GET /v1/modules/[id]/user-stories`, `GET/PATCH/DELETE /v1/user-stories/[id]` |
| **UI** | Project requirement views (story list/detail) |
| **Users** | Member+ |
| **Gate** | `draft → ready_to_test` requires ≥1 active AC (45xxx error otherwise); auto-revert on last-AC archive |
| **Evidence** | `modules/[id]/user-stories/route.ts`, `user-stories/[id]/route.ts`; `lib/user-stories/validation.test.ts` |

**Capabilities:**
- [x] Full CRUD + status gate enforced in DB
- [x] Soft-archive delete

---

### Domain: Acceptance Criteria

#### Feature: AC management

| Aspect | Value |
|---|---|
| **ID** | FEAT-AC-001 |
| **Status** | Stable |
| **Endpoints** | `POST/GET /v1/user-stories/[id]/acceptance-criteria`, `GET/PATCH/DELETE /v1/acceptance-criteria/[id]` |
| **UI** | Story detail (positioned AC list) |
| **Users** | Member+ |
| **Evidence** | `user-stories/[id]/acceptance-criteria/route.ts`, `acceptance-criteria/[id]/route.ts`; `lib/acceptance-criteria/validation.test.ts` |

**Capabilities:**
- [x] Positioned list CRUD; archiving last AC auto-reverts story to `draft` (DB trigger)

---

### Domain: ATC (Automated Test Case)

#### Feature: ATC authoring

| Aspect | Value |
|---|---|
| **ID** | FEAT-ATC-001 |
| **Status** | Stable |
| **Endpoints** | `POST /v1/atcs`, `PATCH /v1/atcs/[id]` |
| **UI** | `(app)/projects/[projectSlug]/atcs/new/page.tsx`, `atcs/[atcId]/page.tsx` (builder, Monaco editor) |
| **Users** | Member+ |
| **Evidence** | `atcs/route.ts`, `atcs/[id]/route.ts`; `lib/atcs/validation.test.ts`, `builder-guards.test.ts`, `optimistic-lock.test.ts`, `app/api/v1/atcs/route-forwarding.test.ts` |

**Capabilities:**
- [x] Create + edit with `version` optimistic lock, tags (≤10), steps + assertions lists
- [x] Validation, sanitize, errors — all unit-tested
- [ ] **No `GET /atcs/[id]` detail endpoint** — reads go through search RPC (⚠️ Read partial)
- [ ] **No `DELETE /atcs/[id]`** — `test_steps` FK is RESTRICT (ATC in use cannot be removed)

#### Feature: Search & discovery

| Aspect | Value |
|---|---|
| **ID** | FEAT-ATC-002 |
| **Status** | Stable |
| **Endpoints** | `GET /v1/atcs/search`, `GET /v1/tests/search`, `GET /v1/search` (global) |
| **UI** | ATC list filters, test list, global search |
| **Users** | Member+ (RLS-scoped) |
| **Evidence** | `atcs/search/route.ts`, `tests/search/route.ts`, `search/route.ts`; migrations `0081`/`0082` (tsv + grants); `lib/atcs/search-contract.test.ts`, `search-isolation.test.ts`, `lib/search/workspace-search-isolation.test.ts` |

**Capabilities:**
- [x] Full-text tsv search + list filters + isolation-tested

#### Feature: Status tracking (template status field)

| Aspect | Value |
|---|---|
| **ID** | FEAT-ATC-003 |
| **Status** | Stable (field) — **effectively dead** |
| **Endpoints** | none (no writer) |
| **UI** | (unverified — any status display would be stale) |
| **Evidence** | `atcs.status` CHECK `pass\|fail\|blocked\|skipped\|running\|unrun` — **dead column** per migration `0050`; data map §271, §654 |

**Capabilities:**
- [ ] `atcs.status` has **no production writer** — live verdicts come from `run_atcs.verdict` (see FEAT-RUN-002)
- [ ] Team must confirm: deprecate column or wire a writer → Discovery Gap §9-1

#### Feature: Classification (technique / priority / layer)

| Aspect | Value |
|---|---|
| **ID** | FEAT-ATC-004 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/atcs` + `PATCH /v1/atcs/[id]` (classification fields), classification API |
| **UI** | ATC builder classification controls (BK-399) |
| **Users** | Member+ |
| **Evidence** | Migrations `0087`/`0088`; `lib/atcs/classification-round-trip.test.ts`, `classification-rpc.test.ts`, `classification-validation.test.ts`, `app/api/v1/atcs/classification-api.test.ts` |

**Capabilities:**
- [x] Layer (`UI|API|Unit`) + technique + priority persisted and validated

#### Feature: Duplicate & usage

| Aspect | Value |
|---|---|
| **ID** | FEAT-ATC-005 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/atcs/[id]/duplicate`, `GET /v1/atcs/[id]/usage` |
| **UI** | ATC detail actions (duplicate), usage panel |
| **Users** | Member+ |
| **Evidence** | `atcs/[id]/duplicate/route.ts` (`route.test.ts`), `atcs/[id]/usage/route.ts`; `lib/atcs/duplicate-rpc.test.ts`, `usage-rpc.test.ts` |

**Capabilities:**
- [x] Duplicate ATC (RPC), query which tests reference it

#### Feature: Export

| Aspect | Value |
|---|---|
| **ID** | FEAT-ATC-006 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/projects/[id]/atcs/export` |
| **UI** | ATC list export action |
| **Users** | Member+ |
| **Evidence** | `projects/[id]/atcs/export/route.ts` (`route.test.ts`); `lib/atcs/csv-export.test.ts`, `export-query.test.ts` |

**Capabilities:**
- [x] CSV export filtered to project

---

### Domain: Test Definitions (ATC chains)

#### Feature: Test suite management

| Aspect | Value |
|---|---|
| **ID** | FEAT-TEST-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET/POST /v1/tests`, `GET /v1/tests/[id]`, `GET /v1/tests/[id]/runs`, `PATCH /v1/tests/[id]/reorder`, `PUT /v1/tests/[id]/tags` |
| **UI** | `(app)/projects/[projectSlug]/tests/` list + `new` + `[testId]` detail |
| **Users** | Member+ (workspace-scoped chain, `version` optimistic lock) |
| **Evidence** | `tests/route.ts`, `tests/[id]/route.ts`; migrations `0026`/`0030`; `lib/tests/validation.test.ts`, `reorder.test.ts`, `tags.test.ts`, `read-isolation.test.ts` |

**Capabilities:**
- [x] Create chain, read detail, reorder steps, tags, run history
- [ ] **No `PATCH` for title/content** (only reorder/tags) — update ⚠️
- [ ] **No `DELETE /tests/[id]`** — delete ❌ → Discovery Gap §9-5

---

### Domain: Test Plans

#### Feature: Plan CRUD & close

| Aspect | Value |
|---|---|
| **ID** | FEAT-TPLAN-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET/POST /v1/projects/[id]/test-plans`, `PATCH /v1/test-plans/[id]` (rename/close) |
| **UI** | `(app)/projects/[projectSlug]/plans/` list + `[planId]` detail |
| **Users** | Member+ (writes via ADR-0012 SECURITY DEFINER RPCs, SELECT-only RLS) |
| **Evidence** | `projects/[id]/test-plans/route.ts`, `test-plans/[id]/route.ts`; migration `0073`; `lib/test-plans/validation.test.ts`, `test-plan-rpc-isolation.test.ts` |

**Capabilities:**
- [x] Create, read, rename, `open → closed`
- [ ] **No `DELETE /test-plans/[id]`** (close only) — delete ⚠️ by design? → Discovery Gap §9-7

#### Feature: Plan ↔ Test linkage

| Aspect | Value |
|---|---|
| **ID** | FEAT-TPLAN-002 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET/POST /v1/test-plans/[id]/tests`, `DELETE /v1/test-plans/[id]/tests/[testId]` |
| **UI** | Plan detail (attach/detach tests) |
| **Evidence** | `test-plans/[id]/tests/route.ts` (`route.test.ts`), `tests/[testId]/route.ts`; migration `0076` (DB `unique(test_plan_id, test_id)` backstops Idempotency-Key) |

**Capabilities:**
- [x] Attach, list, detach tests; idempotent attach

---

### Domain: Test Runs

#### Feature: Run lifecycle

| Aspect | Value |
|---|---|
| **ID** | FEAT-RUN-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/runs` (24h `(test_id, start_token)` idempotency), `GET /v1/runs/[id]`, `POST /v1/runs/[id]/finish {passed\|failed}`, `POST /v1/runs/[id]/abort {reason}`, pg_cron sweep (idle >4 min → `via='sweep'`) |
| **UI** | `tests/[testId]/runs/page.tsx`, `runs/` list + `runs/[runId]` runner |
| **Users** | Member+ |
| **Evidence** | `runs/route.ts` (`route.test.ts`), `runs/[id]/route.ts`, `finish`, `abort`; migrations `0067`; `lib/runs/start-run.test.ts`, `inactivity-sweep-isolation.test.ts` |

**Capabilities:**
- [x] Start (snapshot), finish pass/abort with reason, inactivity sweep
- [x] Leftover `pending` run_steps force-`skipped` on finish; `activity_log run.started/finished`

#### Feature: Step marking & verdict computation

| Aspect | Value |
|---|---|
| **ID** | FEAT-RUN-002 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/runs/[id]/steps/[stepId]/mark` |
| **UI** | Run runner (step verdict buttons: passed/failed/blocked/skipped) |
| **Users** | Member+ (executor) |
| **Evidence** | `runs/[id]/steps/[stepId]/mark/route.ts`; `lib/runs/mark-step.test.ts`, `mark-step-view.test.ts`; data map flow §376 |

**Capabilities:**
- [x] Mark one run_step → recompute parent `run_atcs.verdict` (`failed > blocked > pending > passed`)
- [x] Content frozen at snapshot — ATC edits never rewrite run history

#### Feature: Run Realtime

| Aspect | Value |
|---|---|
| **ID** | FEAT-RUN-003 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | Supabase Realtime (`run_atcs`, `run_steps` in `supabase_realtime` publication) |
| **UI** | Run runner (live updates) |
| **Users** | Viewer of an active run |
| **Evidence** | `lib/runs/realtime-run-channel.test.ts`; migration realtime publication |

**Capabilities:**
- [x] Live verdict push to subscribed clients

#### Feature: Run project report

| Aspect | Value |
|---|---|
| **ID** | FEAT-RUN-004 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/projects/[id]/runs/report` |
| **UI** | `(app)/projects/[projectSlug]/metrics/page.tsx` (run reporting views) |
| **Users** | Member+ |
| **Evidence** | `projects/[id]/runs/report/route.ts`; `lib/runs/report-rpc.test.ts`, `report-isolation.test.ts`, `report-validation.test.ts` |

**Capabilities:**
- [x] Aggregated run report per project, isolation-tested

---

### Domain: Bugs

#### Feature: Bug lifecycle

| Aspect | Value |
|---|---|
| **ID** | FEAT-BUG-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/bugs` (only status entry = `open`, via `bunkai_create_bug`), `GET /v1/bugs`, `GET /v1/bugs/[id]`, `POST /v1/bugs/[id]/status` (forward-only `open → in_progress → resolved → closed`) |
| **UI** | `(app)/projects/[projectSlug]/bugs/` list + `[bugId]` detail (report composer pre-populated from failed run step) |
| **Users** | Member+ |
| **Evidence** | `bugs/route.ts` (`route.test.ts`), `bugs/[id]/route.ts`, `bugs/[id]/status/route.test.ts`; migration `0061` (never archived); `lib/bugs/*` (validation, isolation, list, detail, transition) |

**Capabilities:**
- [x] File with provenance (`run_id`, `run_step_id`, `atc_id` frozen; `SET NULL` upstream), P1–P4, ≤10 evidence URLs
- [x] Status transitions; reopening DB-blocked (regression = new bug)
- [ ] **No general `PATCH /bugs/[id]`** (edit title/description) — update ⚠️ via status/assign only
- [ ] No delete (by design — project only grows)

#### Feature: Assignment

| Aspect | Value |
|---|---|
| **ID** | FEAT-BUG-002 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/bugs/[id]/assign`, `GET /v1/projects/[id]/bugs` (filtered lists) |
| **UI** | Bug detail (assignee picker), bug list |
| **Evidence** | `bugs/[id]/assign/route.ts` (`route.test.ts`); `lib/bugs/assign-bug-isolation.test.ts` |

**Capabilities:**
- [x] Assign/unassign member; notify reporter+assignee (minus actor) on status change

#### Feature: Defect heatmap

| Aspect | Value |
|---|---|
| **ID** | FEAT-BUG-003 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/projects/[id]/bugs/heatmap` |
| **UI** | Metrics page heatmap view |
| **Evidence** | `projects/[id]/bugs/heatmap/route.ts`; `lib/metrics/defect-heatmap.test.ts`, `defect-heatmap-isolation.test.ts` |

**Capabilities:**
- [x] Module × severity distribution

---

### Domain: Milestones

#### Feature: Milestone CRUD

| Aspect | Value |
|---|---|
| **ID** | FEAT-MS-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET/POST /v1/projects/[id]/milestones`, `PATCH /v1/milestones/[id]` |
| **UI** | `(app)/projects/[projectSlug]/milestones/` list + `[milestoneId]` detail (countdown) |
| **Users** | Member+ |
| **Evidence** | `projects/[id]/milestones/route.ts`, `milestones/[id]/route.ts`; migration `0064`; `lib/milestones/countdown.test.ts`, `validation.test.ts`, `milestone-rpc-isolation.test.ts` |

**Capabilities:**
- [x] Create (name 1–100, `target_date` today…+5y), read, edit
- [ ] **No delete / no status / no completion — deferred by design** (data map) → confirm intent §9

---

### Domain: Notifications

#### Feature: Inbox (in-app)

| Aspect | Value |
|---|---|
| **ID** | FEAT-NTF-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/workspaces/[id]/notifications`, `POST /v1/notifications/[id]/read`, `POST /v1/workspaces/[id]/notifications/read-all`, Realtime channel |
| **UI** | `(app)/settings/notifications/page.tsx`, inbox badge (app shell) |
| **Evidence** | notifications routes (`route.test.ts`, `read-all/route.test.ts`); `lib/notifications/*` (view, group-by-day, realtime-channel, isolation ×4) |

**Capabilities:**
- [x] List, mark read, mark all read, realtime, 90-day visibility, idempotent per `(source_event_id, recipient)` from `activity_log` triggers

#### Feature: Preferences

| Aspect | Value |
|---|---|
| **ID** | FEAT-NTF-002 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET/PATCH /v1/notification-preferences` |
| **UI** | Settings → Notifications (preferences grid) |
| **Evidence** | `notification-preferences/route.ts` (`route.test.ts`); `lib/notification-preferences/grid.test.ts`, `write-path.test.ts`; migration `0062` race-safe upsert |

**Capabilities:**
- [x] Event type × channel matrix; `mentions` locked-on (always delivered)

#### Feature: Daily email digest

| Aspect | Value |
|---|---|
| **ID** | FEAT-NTF-003 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET/POST /v1/admin/send-digest` (08:00 UTC cron) |
| **UI** | (server-side only) |
| **Dependencies** | Resend, pg_cron, `notification_digest_log` |
| **Evidence** | `admin/send-digest/route.ts` (`route-methods.test.ts`); migration `0078`; `lib/notifications/send-digest-run.test.ts`, `digest-grouping.test.ts`, `digest-template.test.ts`, `mail/resend-client.test.ts` |

**Capabilities:**
- [x] One row per user per day (`pending → sent|failed`) — DB-side idempotency against cron retries

---

### Domain: Billing

#### Feature: Billing overview

| Aspect | Value |
|---|---|
| **ID** | FEAT-BILL-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/workspaces/[id]/billing` |
| **UI** | `(app)/settings/billing/page.tsx` |
| **Users** | Owner only (RLS SELECT-owner-only) |
| **Evidence** | `workspaces/[id]/billing/route.ts`; `lib/billing/billing-overview-isolation.test.ts` |

**Capabilities:**
- [x] Plan, seats, checkout state visible to owner

#### Feature: Upgrade checkout (Stripe)

| Aspect | Value |
|---|---|
| **ID** | FEAT-BILL-002 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `POST /v1/workspaces/[id]/billing/checkout`, `POST .../checkout/cancel`, `POST /v1/billing/webhook` (Stripe signature) |
| **UI** | `(app)/settings/billing/upgrade/page.tsx` |
| **Dependencies** | Stripe (checkout sessions, webhooks), `stripe_webhook_events` (PK = event id → duplicate no-op) |
| **Evidence** | checkout/cancel/webhook routes (`checkout.test.ts`, `webhook/route.test.ts`, `checkout/cancel/route.openapi.test.ts`); migrations `0077`; `lib/billing/checkout.test.ts`, `checkout-guards-isolation.test.ts` |

**Capabilities:**
- [x] Session create (`target_plan='cloud'`, `seat_quantity > 0`, one open session per workspace), cancel, webhook apply
- [x] Service-role-only writes

#### Feature: Plan limits enforcement

| Aspect | Value |
|---|---|
| **ID** | FEAT-BILL-003 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | DB trigger on project insert (SQLSTATE `45700`) — no HTTP endpoint |
| **UI** | Error surfaced on project-create form |
| **Evidence** | Migration `0048` plan-limit trigger (3 community / 50 cloud / ∞ enterprise); `lib/billing/plan-tiers.test.ts` (tier names only) |

**Capabilities:**
- [x] Limit enforced in Postgres (not app code)
- [ ] Trigger path itself not unit-covered → ⚠️ QA §8

---

### Domain: Data Export

#### Feature: Workspace data export

| Aspect | Value |
|---|---|
| **ID** | FEAT-EXP-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `POST/GET /v1/workspaces/[id]/data-export`, `GET .../data-export/download` |
| **UI** | `(app)/settings/data-export/page.tsx` |
| **Evidence** | data-export routes; migration `0083`; `workspace_exports` state machine `queued → running → completed|failed`, 168h expiry; `lib/workspace-export/authorize.test.ts`, `build-archive.test.ts`, `collect.test.ts` |

**Capabilities:**
- [x] Request archive → Storage bucket `workspace-exports` → authorized download, 7-day expiry

---

### Domain: Environments

#### Feature: Project environments

| Aspect | Value |
|---|---|
| **ID** | FEAT-ENV-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET/POST /v1/projects/[id]/environments`, `PATCH/DELETE /v1/environments/[id]` |
| **UI** | Project settings (environment management) |
| **Evidence** | environment routes; migration `0031` (`unique(project_id, lower(name))`; seeded only for pre-existing projects); `lib/environments/environments-rpc.test.ts`, `validation.test.ts` |

**Capabilities:**
- [x] Full CRUD
- [ ] New projects get **no seeded environments automatically** (data map) → expected behavior? §9

---

### Domain: Coverage & Traceability

#### Feature: Project coverage report

| Aspect | Value |
|---|---|
| **ID** | FEAT-COV-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/projects/[id]/coverage` |
| **UI** | Project coverage views |
| **Evidence** | `projects/[id]/coverage/route.ts`; `lib/coverage/coverage-view.test.ts`, `coverage-isolation.test.ts`; verdicts read from `run_atcs` (**never** dead `atcs.status`, migration `0050`) |

**Capabilities:**
- [x] Per module/AC coverage via `atc_acceptance_criteria × latest run_atcs`

#### Feature: Workspace coverage rollup

| Aspect | Value |
|---|---|
| **ID** | FEAT-COV-002 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/workspaces/[id]/coverage` |
| **UI** | Home dashboard coverage widget |
| **Evidence** | `workspaces/[id]/coverage/route.ts`; `lib/coverage/coverage-isolation.test.ts` |

**Capabilities:**
- [x] Tenant-level coverage rollup

#### Feature: Story traceability chain

| Aspect | Value |
|---|---|
| **ID** | FEAT-COV-003 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/projects/[id]/traceability` (+ export snapshot via RPC) |
| **UI** | `(app)/projects/[projectSlug]/traceability/page.tsx` |
| **Evidence** | `projects/[id]/traceability/route.ts` (`route.test.ts`); `lib/traceability/chain-view.test.ts`, `story-traceability-isolation.test.ts`, `export-snapshot.test.ts` |

**Capabilities:**
- [x] Story → AC → ATC → run verdict chain; exportable snapshot

---

### Domain: Metrics

#### Feature: Recovery cycles (defect fix time)

| Aspect | Value |
|---|---|
| **ID** | FEAT-MET-001 **(NEW domain)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/projects/[id]/metrics/recovery-cycles` |
| **UI** | `(app)/projects/[projectSlug]/metrics/page.tsx` |
| **Evidence** | `projects/[id]/metrics/recovery-cycles/route.ts`; `lib/metrics/recovery-cycle.test.ts`, `recovery-cycle-isolation.test.ts` |

**Capabilities:**
- [x] open → resolved/closed cycle time per bug

---

### Domain: System / Platform

#### Feature: Activity log

| Aspect | Value |
|---|---|
| **ID** | FEAT-SYS-001 |
| **Status** | Stable |
| **Endpoints** | `GET /v1/activity` — **baseline gap CLOSED** (baseline said "no read endpoint") |
| **UI** | `(app)/activity/page.tsx` |
| **Evidence** | `app/api/v1/activity/route.ts` (`route.test.ts`); allowlisted RPC (not raw table select); `lib/activity/list-activity-isolation.test.ts`, `history-validation.test.ts` |

**Capabilities:**
- [x] Append-only log + read API + UI feed; AFTER INSERT triggers fan out to notifications

#### Feature: Idempotency

| Aspect | Value |
|---|---|
| **ID** | FEAT-SYS-002 |
| **Status** | Stable |
| **Endpoints** | `Idempotency-Key` middleware on POST routes; `idempotency_keys` table |
| **Users** | All API consumers |
| **Evidence** | `lib/api/idempotency.test.ts`; data map §idempotency_keys (`pending → succeeded|failed`, keyed workspace+route) |

**Capabilities:**
- [x] Retried POST never double-applies (24h window on run start; DB backstop on plan-tests)

#### Feature: Feature flags infrastructure

| Aspect | Value |
|---|---|
| **ID** | FEAT-SYS-003 |
| **Status** | Stable (schema) / usage untraced |
| **Endpoints** | none traced |
| **Evidence** | `feature_flags` table present in schema; **no runtime reads found** → Discovery Gap §9-9 |

**Capabilities:**
- [x] Table exists
- [ ] Runtime usage not traced

#### Feature: API docs / health / QA guide

| Aspect | Value |
|---|---|
| **ID** | FEAT-SYS-004 |
| **Status** | Stable |
| **Endpoints** | `GET /v1/health`, `GET /openapi`, `GET /v1` |
| **UI** | `api/docs/page.tsx` (Scalar), `qa/page.tsx`, `about/page.tsx`, `design-tokens/page.tsx` |
| **Evidence** | `health/route.ts`, `openapi/route.ts`; `@scalar/api-reference-react`; `@asteasolutions/zod-to-openapi` |

**Capabilities:**
- [x] Health probe, OpenAPI doc, rendered API reference, QA guide

#### Feature: Global search

| Aspect | Value |
|---|---|
| **ID** | FEAT-SYS-005 **(NEW)** |
| **Status** | Stable |
| **Endpoints** | `GET /v1/search` |
| **UI** | Global search input (app shell) |
| **Evidence** | `search/route.ts`; migration `0081`/`0082` search grants; `lib/search/workspace-search-isolation.test.ts` |

**Capabilities:**
- [x] Cross-entity (ATC/tests/…) RLS-scoped search

---

### Domain: Personal Access Tokens

#### Feature: PAT issue / list / revoke

| Aspect | Value |
|---|---|
| **ID** | FEAT-PAT-001 |
| **Status** | Stable |
| **Endpoints** | `POST/GET /v1/tokens`, `DELETE /v1/tokens/[id]` |
| **UI** | `(app)/settings/tokens/page.tsx` |
| **Users** | Authenticated user; issuance **browser-session only** (never via PAT auth); raw secret in companion table (0086); revoked on workspace deletion/member exit |
| **Evidence** | `tokens/route.ts` (`cookie-only-posture.test.ts`), `tokens/[id]/route.ts`; `lib/api/pat.test.ts`, `lib/tokens/*` |

**Capabilities:**
- [x] Issue (secret shown once), list, revoke

---

### Domain: Jira Import

#### Feature: Jira issue import

| Aspect | Value |
|---|---|
| **ID** | FEAT-JIRA-001 |
| **Status** | **Beta** |
| **Endpoints** | `POST /v1/imports` (create job), `GET /v1/imports/[id]` (poll) |
| **UI** | Import flows (poller UI) |
| **Users** | Member+ |
| **Dependencies** | Jira REST (fetch — no SDK dependency) |
| **Evidence** | `imports/route.ts`, `imports/[id]/route.ts`; `import_jobs` state machine `queued → running → completed|failed` (partial stories remain on failure); `lib/jira/import-runner.test.ts`, `extract-acceptance-criteria.test.ts`, `adf-to-markdown.test.ts` |

**Capabilities:**
- [x] Async import job + poll + AC extraction from ADF
- [ ] Partial-import semantics (accepted) — Beta

---

## 3. CRUD matrix

Legend: ✅ Full · ⚠️ Partial/conditional · ❌ Not available · — N/A (by design)

| Entity | Create | Read | Update | Delete | Evidence |
|---|---|---|---|---|---|
| User | ✅ | ✅ | ⚠️ | ❌ | `auth/signup`, `GET /v1/me`; no user-update/delete endpoint → §9 |
| Workspace | ✅ | ✅ | ✅ | ⚠️ Soft (restore + 30d purge) | `workspaces` routes |
| workspace_members | ✅ (via invite accept) | ✅ | ❌ role change | ⚠️ leave only | `membership` DELETE-only → §9-6 |
| workspace_invites | ✅ | ✅ | ⚠️ re-issue = new send | ✅ revoke | `invites` routes |
| projects | ✅ | ✅ (via workspace) | ❌ | ❌ | no `/projects/[id]` route → §9-4 |
| modules | ✅ | ✅ (tree) | ✅ | ⚠️ archive (`DELETE` + `archived_at`) | `modules/[id]` PATCH,DELETE |
| user_stories | ✅ | ✅ | ✅ | ⚠️ soft-archive | `modules/[id]/user-stories`, `user-stories/[id]` |
| acceptance_criteria | ✅ | ✅ | ✅ | ⚠️ archive (reverts story) | `acceptance-criteria/[id]` |
| atcs | ✅ | ⚠️ via search RPC (no `GET /atcs/[id]`) | ✅ (optimistic lock) | ❌ (FK RESTRICT) | `atcs` POST, `atcs/[id]` PATCH |
| atc_steps / atc_assertions | ✅ (embedded in ATC save) | ✅ (via search) | ✅ (embedded) | ✅ (embedded) | `atcs/[id]` PATCH payload |
| atc_acceptance_criteria | ✅ (embedded) | ✅ | ✅ | ✅ | ATC save payload |
| tests (chains) | ✅ | ✅ | ⚠️ reorder/tags only | ❌ | `tests` routes → §9-5 |
| test_steps | ✅ (chain payload) | ✅ | ⚠️ reorder only | ⚠️ | `tests/[id]/reorder` |
| test_plans | ✅ | ✅ | ✅ (rename/close) | ❌ (close only) | `test-plans` routes → §9-7 |
| test_plan_tests (link) | ✅ | ✅ | — | ✅ | `test-plans/[id]/tests` |
| project_environments | ✅ | ✅ | ✅ | ✅ | `environments` routes |
| runs | ✅ | ✅ | ⚠️ finish/abort transitions | — (immutable) | `runs` routes |
| run_atcs | ✅ (snapshot) | ✅ | ⚠️ verdict recompute | — (immutable) | `runs/[id]/steps/.../mark` |
| run_steps | ✅ (snapshot) | ✅ | ⚠️ verdict mark | — (immutable) | mark endpoint |
| bugs | ✅ | ✅ | ⚠️ status/assign only (no general PATCH) | — (never deleted, by design) | `bugs` routes |
| milestones | ✅ | ✅ | ✅ | ❌ (deferred by design) | `milestones` routes → §9 |
| activity_log | ✅ (triggers) | ✅ | — | — (append-only) | `GET /v1/activity` |
| notifications | ✅ (triggers) | ✅ | ⚠️ read flags | — (90-day window) | notifications routes |
| notification_preferences | ✅ upsert | ✅ | ✅ (PATCH upsert) | ⚠️ reset unverified | `notification-preferences` |
| access_tokens | ✅ | ✅ | — | ✅ | `tokens` routes |
| workspace_exports | ✅ | ✅ | — | ⚠️ 168h expiry | `data-export` routes |
| workspace_deletions | ✅ (RPC) | ⚠️ (RPC only, zero RLS policies) | — | — | `0084` |
| billing_checkout_sessions | ✅ | ✅ (owner) | — | — | billing checkout routes |
| stripe_webhook_events | ✅ (webhook) | — | — | — | `billing/webhook` |
| import_jobs | ✅ | ✅ | ⚠️ progress | — | `imports` routes |
| idempotency_keys | ✅ (middleware) | ✅ | — | ⚠️ TTL unverified | `lib/api/idempotency` |
| feature_flags | — | — | — | — | untraced → §9-9 |
| user_view_state | — | — | ⚠️ partial write path | — | data map → §9-10 |
| magic_link_tokens | ✅ (auto) | — | — | ✅ single-use | 0086 companion secret |
| notification_digest_log | ✅ (cron) | ✅ | `pending → sent\|failed` | — | `0078` |

**CRUD gaps summary (feature-impacting):** projects update/delete ❌ · tests update (content)/delete ❌ · atc detail-read ⚠️ + delete ❌ · test plan delete ❌ · member role-change/removal ❌ · user update/delete ❌ · bugs general edit ⚠️ · milestone delete ❌ (by design).

---

## 4. API endpoint inventory

Auth legend: `public` = no session · `session` = cookie JWT or PAT · `webhook` = Stripe signature · `cron` = admin secret. All `session` endpoints are workspace/RLS-scoped.

### Authentication

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| POST | `/v1/auth/signup` | Create account | public |
| POST | `/v1/auth/signin` | Credential sign-in | public |
| POST | `/v1/auth/magic-link` | Request magic link | public |
| POST | `/v1/auth/check-email` | Email status probe | public |
| POST | `/v1/auth/confirm` | Confirm email | public |
| POST | `/v1/auth/resend` | Resend confirmation | public |
| GET | `/v1/me` | Current user + role | session |
| POST | `/v1/me/active-workspace` | Switch active workspace | session |

### Workspace & membership

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET/POST | `/v1/workspaces` | List mine / create | session |
| GET/PATCH/DELETE | `/v1/workspaces/[id]` | Detail / update / soft-delete | session |
| POST | `/v1/workspaces/[id]/restore` | Restore soft-deleted | session |
| DELETE | `/v1/workspaces/[id]/membership` | Leave workspace | session |
| GET/POST | `/v1/workspaces/[id]/invites` | List / create invite (role) | session |
| POST/DELETE | `/v1/workspaces/[id]/invites/[inviteId]` | Re-issue / revoke | session |
| POST | `/v1/invites/accept` | Accept invite | session |
| POST | `/v1/workspaces/[id]/projects` | Create project (plan-capped) | session |
| GET | `/v1/workspaces/[id]/recent-projects` | Recent projects | session |

### Home dashboard

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET | `/v1/workspaces/[id]/active-runs` | Widget: active runs | session |
| GET | `/v1/workspaces/[id]/open-bugs` | Widget: open bugs | session |
| GET | `/v1/workspaces/[id]/coverage` | Widget: coverage | session |
| GET | `/v1/activity` | Activity feed | session |

### Modules & requirements

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| POST | `/v1/projects/[id]/modules` | Create root module | session |
| PATCH/DELETE | `/v1/modules/[id]` | Edit / archive module | session |
| POST/GET | `/v1/modules/[id]/user-stories` | Create / list stories | session |
| GET/PATCH/DELETE | `/v1/user-stories/[id]` | Story CRUD | session |
| POST/GET | `/v1/user-stories/[id]/acceptance-criteria` | Create / list ACs | session |
| GET/PATCH/DELETE | `/v1/acceptance-criteria/[id]` | AC CRUD | session |

### ATCs

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| POST | `/v1/atcs` | Create ATC | session |
| PATCH | `/v1/atcs/[id]` | Edit (optimistic lock, classification) | session |
| POST | `/v1/atcs/[id]/duplicate` | Duplicate | session |
| GET | `/v1/atcs/[id]/usage` | Which tests use it | session |
| GET | `/v1/atcs/search` | Full-text + filters | session |
| GET | `/v1/projects/[id]/atcs/export` | CSV export | session |

### Tests (chains)

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET/POST | `/v1/tests` | List / create | session |
| GET | `/v1/tests/[id]` | Detail | session |
| GET | `/v1/tests/[id]/runs` | Run history | session |
| PATCH | `/v1/tests/[id]/reorder` | Reorder steps | session |
| PUT | `/v1/tests/[id]/tags` | Replace tags | session |
| GET | `/v1/tests/search` | Full-text search | session |

### Test plans

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET/POST | `/v1/projects/[id]/test-plans` | List / create | session |
| PATCH | `/v1/test-plans/[id]` | Rename / close | session |
| GET/POST | `/v1/test-plans/[id]/tests` | List / attach test | session |
| DELETE | `/v1/test-plans/[id]/tests/[testId]` | Detach test | session |

### Runs

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| POST | `/v1/runs` | Start run (24h idempotency) | session |
| GET | `/v1/runs/[id]` | Run detail (snapshots) | session |
| POST | `/v1/runs/[id]/finish` | Finish `passed\|failed` | session |
| POST | `/v1/runs/[id]/abort` | Abort with reason | session |
| POST | `/v1/runs/[id]/steps/[stepId]/mark` | Mark step → verdict | session |
| GET | `/v1/projects/[id]/runs/report` | Project run report | session |

### Bugs

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET/POST | `/v1/bugs` | List / file (only `open` entry) | session |
| GET | `/v1/bugs/[id]` | Detail | session |
| POST | `/v1/bugs/[id]/status` | Forward-only transition | session |
| POST | `/v1/bugs/[id]/assign` | Assign | session |
| GET | `/v1/projects/[id]/bugs` | Project-filtered list | session |
| GET | `/v1/projects/[id]/bugs/heatmap` | Severity × module | session |

### Milestones

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET/POST | `/v1/projects/[id]/milestones` | List / create | session |
| PATCH | `/v1/milestones/[id]` | Edit | session |

### Notifications

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET | `/v1/workspaces/[id]/notifications` | Inbox list | session |
| POST | `/v1/notifications/[id]/read` | Mark read | session |
| POST | `/v1/workspaces/[id]/notifications/read-all` | Mark all read | session |
| GET/PATCH | `/v1/notification-preferences` | Preferences matrix | session |
| GET/POST | `/v1/admin/send-digest` | Daily digest (cron) | cron |

### Billing

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET | `/v1/workspaces/[id]/billing` | Overview (owner) | session |
| POST | `/v1/workspaces/[id]/billing/checkout` | Stripe checkout (cloud) | session |
| POST | `/v1/workspaces/[id]/billing/checkout/cancel` | Cancel session | session |
| POST | `/v1/billing/webhook` | Stripe events | webhook |

### Export / environments / metrics / coverage

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| POST/GET | `/v1/workspaces/[id]/data-export` | Request / status | session |
| GET | `/v1/workspaces/[id]/data-export/download` | Download archive | session |
| GET/POST | `/v1/projects/[id]/environments` | List / create | session |
| PATCH/DELETE | `/v1/environments/[id]` | Edit / delete | session |
| GET | `/v1/projects/[id]/coverage` | Coverage report | session |
| GET | `/v1/projects/[id]/traceability` | Story chain | session |
| GET | `/v1/projects/[id]/metrics/recovery-cycles` | Defect recovery time | session |

### System / tokens / import / search / docs

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| GET | `/v1/health` | Health probe | public |
| GET | `/v1`, `/openapi`, `/api/openapi` | OpenAPI documents | public |
| GET | `/v1/search` | Global search | session |
| POST/GET | `/v1/tokens` | Issue / list PAT | session |
| DELETE | `/v1/tokens/[id]` | Revoke PAT | session |
| POST | `/v1/imports` | Start Jira import | session |
| GET | `/v1/imports/[id]` | Poll job | session |

**Route count:** 94 `route.ts` files. **No endpoint found for:** project update/delete, test content update/delete, test-plan delete, member role-change, admin member removal, user update/delete, ATC detail GET, ATC delete, bug general edit (see §3 gaps).

---

## 5. UI component inventory

### Forms

| Form | Page | Feature |
|---|---|---|
| Sign-in / Sign-up / Magic link / Email-resend | `(auth)/login` | AUTH-001/002/003/005 |
| Onboarding (first workspace) | `(app)/onboarding` | AUTH-001 / WS-001 |
| Workspace create/edit | `(app)/settings/workspaces` | WS-001 |
| Invite member (role select) | `(app)/workspaces/[id]/members` | WS-002 |
| Accept invite | `invites/accept` | WS-002 |
| Project create | `(app)/projects/new` | PRJ-001 |
| ATC builder (Monaco, steps/assertions, classification) | `(app)/.../atcs/new`, `atcs/[atcId]` | ATC-001/004/005 |
| Test chain builder (+ reorder, tags) | `(app)/.../tests/new`, `tests/[testId]` | TEST-001 |
| Test plan create/attach | `(app)/.../plans`, `plans/[planId]` | TPLAN-001/002 |
| Bug report composer (evidence URLs, provenance pre-fill) | `(app)/.../bugs/[bugId]` | BUG-001 |
| Milestone create/edit | `(app)/.../milestones`, `[milestoneId]` | MS-001 |
| Environment create/edit | project settings | ENV-001 |
| Billing upgrade (seats, plan) | `(app)/settings/billing/upgrade` | BILL-002 |
| Data export request | `(app)/settings/data-export` | EXP-001 |
| Notification preferences grid | `(app)/settings/notifications` | NTF-002 |
| PAT issue | `(app)/settings/tokens` | PAT-001 |

### Dashboards / views

| View | Page | Feature |
|---|---|---|
| Home (4 widgets + welcome) | `(app)/home` | HOME-001 |
| Activity feed | `(app)/activity` | SYS-001 |
| Project overview | `(app)/.../[projectSlug]` | PRJ-001 |
| ATC list + search/filters | ATC tab | ATC-002 |
| Test list | tests tab | TEST-001 |
| Run list + runner (realtime) | `runs`, `runs/[runId]` | RUN-001/002/003 |
| Plans, bugs list/detail, milestones | respective tabs | TPLAN/BUG/MS |
| Traceability chain | `traceability` | COV-003 |
| Metrics (coverage, heatmap, recovery, runs report) | `metrics` | COV-001, BUG-003, MET-001, RUN-004 |
| Settings (account, billing, tokens, notifications, workspaces, data-export) | `(app)/settings/*` | platform features |
| Workspace restore | `workspaces/[id]/restore` | WS-003 |
| API docs (Scalar) / QA guide / About / Design tokens | `api/docs`, `qa`, `about`, `design-tokens` | SYS-004 |

### Actions (modals, dialogs, confirmations)

| Action | Feature |
|---|---|
| Duplicate ATC, ATC CSV export | ATC-005, ATC-006 |
| Mark step verdict (passed/failed/blocked/skipped), finish run, abort run | RUN-001/002 |
| Bug status transition, assign | BUG-001/002 |
| Plan close, attach/detach test | TPLAN-001/002 |
| Invite re-issue / revoke, leave workspace | WS-002/004 |
| Workspace delete (confirm) / restore | WS-003 |
| PAT revoke | PAT-001 |
| Notifications read / read-all | NTF-001 |

---

## 6. Third-party integrations

| Service | Purpose | Package | Status | Features using it |
|---|---|---|---|---|
| Supabase (Postgres, Auth, Storage, Realtime, RLS, RPC) | Primary DB + auth + realtime + file storage | `@supabase/supabase-js`, `@supabase/ssr` | Active | ALL |
| Stripe | Checkout sessions, webhooks, plan state | `stripe` | Active | BILL-001/002/003 |
| Resend | Email: magic link, confirmation, daily digest, deletion notice | `resend` (`lib/mail/resend-client`) | Active (invite email NOT wired → §9-2) | AUTH-003/005, NTF-003, WS-003 |
| Jira Cloud REST | Issue import (ADF → markdown) | fetch (`lib/jira`, no SDK) | Active — **Beta** | JIRA-001 |
| Vercel | Hosting / auto-deploy | — | Active | system (per Phase 1 assessment) |
| pg_cron | Run inactivity sweep, digest scheduling | SQL (migrations) | Active | RUN-001, NTF-003 |
| Scalar | Rendered API reference | `@scalar/api-reference-react` | Active | SYS-004 |
| Zod + zod-to-openapi | Validation + OpenAPI generation | `zod`, `@asteasolutions/zod-to-openapi` | Active | SYS-004, all contracts |

---

## 7. Feature flags and WIP

### Feature flags

| Flag | Description | Default | Environment |
|---|---|---|---|
| `feature_flags` (DB table) | Schema present; **runtime reads untraced — no flag evaluated in app code found** | — | — |
| `FEATURE_TICKS` | **Not a flag** — login-page marketing copy | — | — |

No `ENABLE_*` / `BETA_*` / `isFeatureEnabled` patterns found. → §9-9.

### Planned / deferred / WIP

| Planned feature | Evidence (TODOs, stubs, design notes) | Estimated status |
|---|---|---|
| Milestone delete + status/completion | Data map: "No status, no completion, no deletion — deferred by design" | Planned (deferred) |
| Member role change / admin member removal | `membership` route is DELETE-only; no admin endpoints | Planned (gap) |
| Ownership transfer endpoint | Data map: "ownership is transferred, never invited" — no route found | Unverified |
| Invite email delivery | No Resend call in invites flow | Planned (copy-link today) |
| Project update/delete | No `/projects/[id]` route file | Planned / intentional? |
| Test chain content edit / delete | Only reorder/tags PATCH/PUT | Planned / intentional? |
| Test plan delete | Close only | Planned / intentional? |
| `atcs.status` writer | Dead column since `0050` | Deprecated or Planned (needs decision) |
| `user_view_state` write path | Data map: "write path verified only partially" | WIP |
| ComingSoon UI placeholders | None found (only former instance: settings/tokens, already replaced — BK-87) | — |

---

## 8. QA relevance

**Feature test coverage matrix.**
- **Unit** = target-repo `bun test` (`lib/**`, `app/api/**/route.test.ts` — ~190 files, read-only inventory)
- **Integration** = QA repo `tests/integration/` (3 real files: `auth/user-session`, `module-example` ×2)
- **E2E** = QA repo `tests/e2e/` + `kata-manifest.json` (9 ATCs: AuthApi 2, LoginPage 2, Example* boilerplate 5)

| Feature ID | Unit | Integration | E2E | Status |
|---|---|---|---|---|
| FEAT-AUTH-001 | ✅ | ✅ | ⚠️ | E2E only as setup |
| FEAT-AUTH-002 | ✅ | ✅ | ✅ | KATA-covered |
| FEAT-AUTH-003 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-AUTH-004 | ✅ | ✅ | ⚠️ | OK |
| FEAT-AUTH-005 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-WS-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-WS-002 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-WS-003 | ✅ | ❌ | ❌ | Needs Int/E2E (high risk) |
| FEAT-WS-004 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-HOME-001 | ❌ | ❌ | ⚠️ | Needs Unit |
| FEAT-PRJ-001 | ✅ | ❌ | ⚠️ | Needs Int |
| FEAT-MOD-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-US-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-AC-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-ATC-001 | ✅ | ❌ | ❌ | Needs Int/E2E (high risk) |
| FEAT-ATC-002 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-ATC-003 | ❌ | ❌ | ❌ | Dead column — decision first |
| FEAT-ATC-004 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-ATC-005 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-ATC-006 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-TEST-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-TPLAN-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-TPLAN-002 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-RUN-001 | ✅ | ❌ | ❌ | Needs Int/E2E (high risk) |
| FEAT-RUN-002 | ✅ | ❌ | ❌ | Needs Int/E2E (high risk) |
| FEAT-RUN-003 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-RUN-004 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-BUG-001 | ✅ | ❌ | ❌ | Needs Int/E2E (high risk) |
| FEAT-BUG-002 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-BUG-003 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-MS-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-NTF-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-NTF-002 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-NTF-003 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-BILL-001 | ✅ | ❌ | ❌ | Needs Int/E2E (high risk) |
| FEAT-BILL-002 | ✅ | ❌ | ❌ | Needs Int/E2E (HIGH risk) |
| FEAT-BILL-003 | ⚠️ | ❌ | ❌ | DB trigger uncovered |
| FEAT-EXP-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-ENV-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-COV-001 | ✅ | ❌ | ❌ | Needs Int/E2E (high risk) |
| FEAT-COV-002 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-COV-003 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-MET-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-SYS-001 | ✅ | ❌ | ❌ | Needs Int |
| FEAT-SYS-002 | ✅ | ❌ | ❌ | Needs Int |
| FEAT-SYS-003 | ❌ | ❌ | ❌ | Untraced (§9-9) |
| FEAT-SYS-004 | ❌ | ❌ | ❌ | Low risk (docs/health) |
| FEAT-SYS-005 | ✅ | ❌ | ❌ | Needs Int |
| FEAT-PAT-001 | ✅ | ❌ | ❌ | Needs Int/E2E |
| FEAT-JIRA-001 | ✅ | ❌ | ❌ | Beta — needs E2E |

**Totals:** Unit ✅ 44 · ⚠️ 1 · ❌ 5 · Integration ✅ 3 (auth only) · E2E ✅ 1 · ⚠️ 4.

**High-risk features** (prioritize automation):

| Feature | Risk | Reason |
|---|---|---|
| FEAT-BILL-002 | HIGH | Revenue — Stripe money path, webhook idempotency |
| FEAT-BILL-003 | HIGH | Plan caps enforced only by DB trigger (SQLSTATE 45700) |
| FEAT-AUTH-001/002/003/004 | HIGH | Security — session, PAT coexistence, magic link |
| FEAT-RUN-001/002 | HIGH | Verdict integrity — every coverage/traceability report depends on `run_atcs` |
| FEAT-COV-001/003 | HIGH | QA decision data; reads must never touch dead `atcs.status` |
| FEAT-ATC-001 | HIGH | Core artifact; optimistic lock + builder validation |
| FEAT-WS-003 | HIGH | Irreversible purge after 30-day window |
| FEAT-WS-002 | HIGH | Role escalation surface (`admin` invite) |
| FEAT-BUG-001 | MEDIUM-HIGH | Provenance frozen at filing; forward-only status |
| FEAT-NTF-003 | MEDIUM | Email spam risk if digest idempotency breaks |

---

## 9. Discovery gaps (MANDATORY)

1. **`atcs.status` dead column** — no production writer since migration `0050`; live verdicts come from `run_atcs`. Is FEAT-ATC-003 deprecated (remove column/UI) or planned (wire a writer)? *Team decision required.*
2. **Invite email delivery not wired** — no Resend call in `workspaces/[id]/invites` route or `lib/workspaces/invites.ts`. Copy-link flow assumed; confirm whether invite emails are Planned or intentionally omitted.
3. **Missing migrations `0079`, `0080`** — sequence jumps `0078 → 0081`. Are they reverted/deleted, or absent from repo?
4. **No project update/delete endpoints** — no `app/api/v1/projects/[id]/route.ts`. Intentional or WIP? UI has no project edit/delete screen either.
5. **No test-chain content update / delete** — only `reorder` + `tags`. `test_steps` FK is RESTRICT; confirm intended lifecycle.
6. **No member role-change / admin removal endpoints** — `workspaces/[id]/membership` is DELETE-only (leave). How does an owner change a member's role today? (Invites carry roles — re-invite?) *Ownership-transfer mechanism also not found.*
7. **No test-plan delete** — close only. By design?
8. **Milestone delete/status deferred by design** (data map) — confirm this is still intended; no discovery needed if confirmed.
9. **`feature_flags` table untraced** — no runtime reads found; table may be dead infra.
10. **`user_view_state` write path partially verified** (data map note) — which views persist?
11. **DBHub MCP unavailable this session** — schema derived from migrations `0001–0088` only; live-DB checks (actual RLS policies, trigger behavior, row counts) NOT performed. → re-verify when DBHUB_* creds configured.
12. **Target-repo unit tests (~190 files) vs QA-repo coverage** — coverage above mixes both. Confirm whether target `bun test` counts toward ROI/automation scoring in `test-documentation`.
13. **Ownership-transfer endpoint not found** — data map states ownership is transferred, never invited; no route implements transfer. Need endpoint location or gap confirmation.
14. **New projects get no seeded environments** (migration `0031` seeds only pre-existing) — expected product behavior?
15. **No `GET /projects` list endpoint** — project listing appears to come from workspace detail / recent-projects / RSC direct query. Confirm read path for `/projects` page.

### Phase 5 cross-reference with `business-data-map.md`

- **Entities → features:** every core entity maps to ≥1 feature. Orphans flagged: `user_view_state` (no owning feature — §9-10), `qa_inspector` role infrastructure (migrations `0085`/`0086`, no UI/feature surfaced), `feature_flags` (§9-9).
- **Flows → features:** all data-map state machines mapped (story gate → US-001; run lifecycle → RUN-001/002; bug → BUG-001; invite pending → WS-002; export → EXP-001; deletion → WS-003; import → JIRA-001; digest → NTF-003; checkout → BILL-002; plan limit → BILL-003).
- **Features → entities:** FEAT-SYS-004 (docs/health) is intentionally entity-less; FEAT-SYS-003 has a table but no traced runtime usage.
- **Baseline mismatches closed:** baseline claimed "Activity: no read endpoint" → now `GET /v1/activity` exists (FEAT-SYS-001). Baseline had no run/bug/plan/notification/billing model at all → all now first-class.

### Sources

| Source | Extracted | Tool |
|---|---|---|
| `app/api/**/route.ts` (94 files) | Endpoint inventory + verified HTTP methods | Read + grep |
| `app/**/page.tsx` (38 pages) | UI inventory | find |
| `supababase/migrations/0001–0088` | Entity/schema evidence (0079/0080 missing) | Read (DBHub unavailable) |
| `package.json` (target) | Integrations, `bun test` scripts | Read |
| `lib/**`, `app/**/route.test.ts` (~190 unit test files) | Unit coverage column | find |
| `tests/` + `kata-manifest.json` (QA repo) | Integration/E2E/KATA columns | Read |
| `git log --oneline -30` | Recent feature activity (BK-87/399 etc.) | git |
| `.context/business/business-data-map.md` (1024 lines) | Phase 5 cross-ref, flows, design decisions | Read |
| `.context/business/business-feature-map.md` (577 lines, baseline) | FEAT-ID reuse | Read |
| `.context/PRD/`, `.context/SRS/` | Product context | Skim |

---

*CANDIDATE FILE — awaiting user confirmation before replacing `.context/business/business-feature-map.md` (UPDATE mode: never auto-overwrite).*
