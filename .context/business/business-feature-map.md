# Business Feature Map — Bunkai (upex-bunkai-tms)

> Generated: 2026-06-20 | Sources: target repo code scan, Jira PBIs, SRS, PRD, domain glossary

---

## 1. Inventory summary

| Category | Features | Status |
|----------|----------|--------|
| **Core** (authentication, workspace, projects, modules, US, AC, ATCs, tests) | 8 domains | Stable (implemented) |
| **Secondary** (execution runs, defects, coverage/traceability, account/settings) | 4 domains | Stable (implemented) |
| **Beta** (imports/Jira sync, QA guide) | 2 features | Partially implemented |
| **Planned** (SSO OAuth, ATC propagation, tag filtering, activity feed, export snapshots, retained-leak tests) | 6+ features | In development |

---

## 2. Feature catalog (by domain)

### Domain: Authentication & Identity

#### Feature: Email Magic Link Sign-in

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AUTH-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/auth/magic-link`, `POST /api/v1/auth/signup` |
| **UI** | `/login` page (bilateral layout), `/auth/callback` |
| **Users** | Unauthenticated visitors |
| **Dependencies** | Supabase Auth (email OTP), Resend (email delivery) |
| **Evidence** | `app/api/auth/magic-link/route.ts`, `app/(auth)/login/page.tsx` |

**Capabilities:**
- [x] Send magic-link OTP via email
- [x] Headless signup with PAT minting
- [x] Session cookie via Supabase SSR
- [x] `/auth/callback` route handler

#### Feature: PAT (Personal Access Token) Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AUTH-002 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/tokens`, `GET /api/v1/tokens`, `DELETE /api/v1/tokens/[id]` |
| **UI** | Account settings section (BK-88) |
| **Users** | Authenticated users with session |
| **Dependencies** | SHA-256 hashing, Supabase DB |
| **Evidence** | `app/api/tokens/route.ts`, `lib/api/pat.ts` |

**Capabilities:**
- [x] Issue PAT (`bk_pat_*` prefix)
- [x] List own tokens
- [x] Revoke token (soft-delete)
- [ ] Scope-based permission enforcement (partially — scope array exists but enforcement is incomplete per BK-117, BK-134)

#### Feature: OAuth Sign-in (GitHub / Google)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AUTH-003 |
| **Status** | Planned |
| **Endpoints** | N/A (not implemented) |
| **UI** | Not built yet |
| **Users** | Unauthenticated visitors |
| **Dependencies** | Supabase Auth OAuth providers, GitHub OAuth app, Google Cloud OAuth |
| **Evidence** | BK-3 (Story in backlog) |

**Capabilities:**
- [ ] GitHub OAuth button
- [ ] Google OAuth button
- [ ] Account linking with existing email-based accounts

---

### Domain: Workspace Management

#### Feature: Workspace CRUD

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-WS-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/workspaces`, `GET /api/v1/workspaces`, `GET /api/v1/workspaces/[id]`, `PATCH /api/v1/workspaces/[id]` |
| **UI** | `/onboarding` (first workspace), workspace switcher in sidebar |
| **Users** | Authenticated users |
| **Dependencies** | Supabase DB, RLS (workspace_members) |
| **Evidence** | `app/api/workspaces/route.ts`, `app/api/workspaces/[id]/route.ts` |

**Capabilities:**
- [x] Create workspace (auto-bootstrap via trigger on auth.users INSERT)
- [x] List workspaces for current user
- [x] Get workspace detail
- [x] Update workspace name/settings
- [x] Switch active workspace (`POST /api/v1/me/active-workspace`)
- [ ] Leave workspace (BK-90 — Story pending)

#### Feature: Workspace Invites & Roles

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-WS-002 |
| **Status** | Stable (with known bugs) |
| **Endpoints** | `POST /api/v1/workspaces/[id]/invites`, `GET /api/v1/workspaces/[id]/invites`, `POST /api/v1/workspaces/[id]/invites/[inviteId]`, `DELETE /api/v1/workspaces/[id]/invites/[inviteId]`, `POST /api/v1/invites/accept` |
| **UI** | `/invites` (pending + inbound), `/members` (member list) |
| **Users** | Workspace admins (create/rotate/revoke), authenticated (accept) |
| **Dependencies** | Resend (invite email), Supabase DB, RLS |
| **Evidence** | `app/api/workspaces/[id]/invites/route.ts`, `app/api/invites/accept/route.ts` |

**Capabilities:**
- [x] Invite member by email with role
- [x] List pending invites
- [x] Rotate invite token
- [x] Revoke invite
- [x] Accept invite with token
- [x] Roles: viewer, member, admin, owner
- [x] Auto-assign admin role to workspace creator
- [x] `bootstrap_first_workspace()` trigger on auth signup

**Known bugs:** BK-60 (no email uniqueness check against active members), BK-61 (duplicate invites allowed), BK-62 (role overwrite on accept via upsert)

---

### Domain: Project & Module Hierarchy

#### Feature: Project CRUD

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-PROJ-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/workspaces/[id]/projects` |
| **UI** | `/projects` listing, project workbench |
| **Users** | Workspace members |
| **Dependencies** | Supabase DB, modules, user_stories, ATCs |
| **Evidence** | `app/api/workspaces/[id]/projects/route.ts` |

**Capabilities:**
- [x] Create project with slug + name inside workspace
- [x] List projects for workspace
- [x] View project workbench (module tree + ATC table)

#### Feature: Module Tree (Nested, 6-level max)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-MOD-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/projects/[id]/modules`, `PATCH /api/v1/modules/[id]`, `DELETE /api/v1/modules/[id]` |
| **UI** | Module tree in project workbench |
| **Users** | Workspace members |
| **Dependencies** | Supabase DB, materialized path, recursive CTEs |
| **Evidence** | `app/api/projects/[id]/modules/route.ts`, `app/api/modules/[id]/route.ts` |

**Capabilities:**
- [x] Create module with name + optional parent
- [x] Rename module
- [x] Move module to different parent (`move_module()` RPC)
- [x] Soft-delete module (cascade to children via `soft_delete_module_cascade()`)
- [x] Auto-computed materialized `path` column (max depth 6)
- [x] Tree view with ACs + ATCs per-module count

**Known bugs:** BK-67 (success toast suppressed at depth >= 5), BK-68 (1-char names pass client-side)

#### Feature: Explorer Views — Tree, Table & Mind Map

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-MOD-002 |
| **Status** | Planned |
| **Endpoints** | N/A (frontend-only) |
| **UI** | Not built yet |
| **Users** | Workspace members |
| **Evidence** | BK-98 (Story in backlog) |

**Capabilities:**
- [ ] Switch between tree / table / mind map views in hardened explorer
- [ ] Tab-based navigation per-module (BK-147)

---

### Domain: User Stories & Acceptance Criteria

#### Feature: User Story Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-US-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/modules/[id]/user-stories`, `GET /api/v1/modules/[id]/user-stories`, `GET /api/v1/user-stories/[id]`, `PATCH /api/v1/user-stories/[id]`, `DELETE /api/v1/user-stories/[id]` |
| **UI** | User story form (create/edit) in project workbench |
| **Users** | Workspace members |
| **Dependencies** | Supabase DB, modules, Jira import (external_id linking) |
| **Evidence** | `app/api/modules/[id]/user-stories/route.ts`, `app/api/user-stories/[id]/route.ts` |

**Capabilities:**
- [x] Create user story anchored to a module
- [x] List user stories per module
- [x] Get user story detail
- [x] Update title, description (Markdown)
- [x] Soft-delete user story
- [x] External ID linking (Jira issue key)
- [x] Partial unique index on `user_stories(external_id)` WHERE NOT NULL
- [x] `ready_to_test` status gate (`set_user_story_ready_to_test()` serialized RPC)

#### Feature: Acceptance Criteria Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AC-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/user-stories/[id]/acceptance-criteria`, `GET /api/v1/user-stories/[id]/acceptance-criteria`, `PATCH /api/v1/acceptance-criteria/[id]`, `DELETE /api/v1/acceptance-criteria/[id]` |
| **UI** | AC list inside user story form |
| **Users** | Workspace members |
| **Dependencies** | Supabase DB, user stories |
| **Evidence** | `app/api/user-stories/[id]/acceptance-criteria/route.ts`, `app/api/acceptance-criteria/[id]/route.ts` |

**Capabilities:**
- [x] Create AC with title, description, position
- [x] List ACs for a user story (ordered by position)
- [x] Update AC
- [x] Soft-delete AC
- [x] Position-based ordering (`reorder_acs()` RPC, partial unique index)

#### Feature: Markdown Editor

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-MD-001 |
| **Status** | Stable (with known bugs) |
| **Endpoints** | N/A (frontend component) |
| **UI** | Markdown editor component in user story form |
| **Users** | Workspace members |
| **Dependencies** | Custom `textarea` + byte counter |
| **Evidence** | Markdown editor component |

**Capabilities:**
- [x] Write Markdown with preview
- [x] Byte counter for content size

**Known bugs:** BK-99 (50 KB limit not enforced server-side), BK-100 (90% warning threshold not implemented)

#### Feature: Jira Import

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-IMP-001 |
| **Status** | Beta |
| **Endpoints** | `POST /api/v1/imports`, `GET /api/v1/imports`, `GET /api/v1/imports/[id]` |
| **UI** | Import page/button (to be built) |
| **Users** | Workspace members |
| **Dependencies** | `jira.js` npm package, Jira REST API, Supabase DB |
| **Evidence** | `app/api/imports/route.ts`, `app/api/imports/[id]/route.ts` |

**Capabilities:**
- [x] Enqueue Jira import job (async)
- [x] List import jobs for workspace
- [x] Poll import job status (SSE-ready)
- [x] Import Jira epics → user stories with external_id linking
- [x] One-active-import constraint per project

**Known bugs:** BK-142 (Jira credentials not configured in staging → import fails with `jira_unauthorized`)

---

### Domain: ATC Library (Acceptance Test Cases)

#### Feature: ATC CRUD

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ATC-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/atcs`, `PATCH /api/v1/atcs/[id]` |
| **UI** | ATC table in project workbench, ATC builder |
| **Users** | Workspace members |
| **Dependencies** | Supabase DB, steps, assertions, AC bindings |
| **Evidence** | `app/api/atcs/route.ts`, `app/api/atcs/[id]/route.ts` |

**Capabilities:**
- [x] Create ATC with steps, assertions, AC bindings (via `create_atc_v1` RPC)
- [x] Edit ATC with optimistic locking (`version` column, full replace)
- [x] ATC layers: UI, API, Unit (tagged with `--layer-ui`, `--layer-api`, `--layer-unit`)
- [x] Steps with position, content, input_data, expected
- [x] Assertions with position, content
- [x] AC-ATC bindings via `atc_acceptance_criteria` table

**Known bugs:** BK-96 (PATCH returns 412 instead of 200 on happy path)

#### Feature: ATC Search & Autocomplete

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ATC-002 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | Search bar (not built) |
| **Users** | Workspace members |
| **Evidence** | BK-20 (Story in backlog) |

**Capabilities:**
- [ ] Search ATCs by title, slug, tags
- [ ] Autocomplete when chaining ATCs into a test

#### Feature: ATC Propagation & Usage Reporting

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ATC-003 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | "Used in N tests" report (BK-22), cascade on edit (BK-21) |
| **Users** | Workspace members |
| **Evidence** | BK-21, BK-22 (Stories in backlog) |

**Capabilities:**
- [ ] Cascade ATC edits to all tests using it
- [ ] "Used in N tests" usage report

#### Feature: ATC Duplicate

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ATC-004 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | Duplicate button (not built) |
| **Users** | Workspace members |
| **Evidence** | BK-23 (Story in backlog) |

**Capabilities:**
- [ ] Duplicate ATC with steps and assertions

---

### Domain: Tests (ATC Chains)

#### Feature: Test Builder (Chain ATCs)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-TEST-001 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | Test builder (not built) |
| **Users** | Workspace members |
| **Dependencies** | ATCs, order/reorder |
| **Evidence** | BK-27, BK-28 (Stories in backlog) |

**Capabilities:**
- [ ] Assemble a test by chaining ATCs in order
- [ ] Reorder ATCs inside a test

#### Feature: Test View & Tags

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-TEST-002 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | Test detail view (not built) |
| **Users** | Workspace members |
| **Evidence** | BK-32, BK-33 (Stories in backlog) |

**Capabilities:**
- [ ] View test with all chained ATCs expanded
- [ ] Assign reserved and custom tags

---

### Domain: Manual Execution & Runs

#### Feature: Run Execution (Manual)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-RUN-001 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | Run execution page (not built) |
| **Users** | Workspace members |
| **Evidence** | BK-34, BK-35, BK-36, BK-37, BK-38, BK-39 (Stories in backlog) |

**Capabilities:**
- [ ] Start a manual run in a chosen environment
- [ ] Mark each step pass / fail / block
- [ ] Abort a run with reason
- [ ] View past runs filtered by outcome
- [ ] Filter project runs with pass/fail totals
- [ ] Finish run with final verdict

---

### Domain: Defect Management & Heatmap

#### Feature: Defect Filing & List

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-DEF-001 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | Defect filing form, defect list, heatmap (not built) |
| **Users** | Workspace members |
| **Evidence** | BK-40, BK-41, BK-42, BK-43 (Stories in backlog) |

**Capabilities:**
- [ ] File a defect from a failing run step
- [ ] List and filter defects by module, status, severity
- [ ] View defect heatmap with week-over-week trend per module
- [ ] Sync defects one-way to external tracker

---

### Domain: Coverage & Traceability

#### Feature: Traceability Chain

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-TRACE-001 |
| **Status** | Planned |
| **Endpoints** | Not implemented |
| **UI** | Traceability view, coverage dashboard, activity feed, export (not built) |
| **Users** | Workspace members |
| **Evidence** | BK-45, BK-46, BK-47, BK-48, BK-49, BK-50 (Stories in backlog) |

**Capabilities:**
- [ ] Render full US → bug evidence chain
- [ ] Surface untested ACs and modules with not-run filter
- [ ] Compute time-to-green per user story
- [ ] Filter chain by verdict, module, date range
- [ ] Activity feed over existing activity_log
- [ ] Export chain as read-only snapshot

---

### Domain: Account & Settings

#### Feature: Account Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ACC-001 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/me` |
| **UI** | Account view (identity, role, sign out), settings hub |
| **Users** | Authenticated users |
| **Dependencies** | Supabase Auth, workspace_members |
| **Evidence** | `app/api/me/route.ts`, account pages |

**Capabilities:**
- [x] View identity + role
- [x] Sign out
- [x] View workspaces I belong to
- [x] Manage PATs (FEAT-AUTH-002)
- [x] Settings hub

#### Feature: Workspace Membership Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ACC-002 |
| **Status** | Stable |
| **Endpoints** | `PATCH /api/v1/me/active-workspace` |
| **UI** | Workspace switcher, leave workspace |
| **Users** | Authenticated users |
| **Dependencies** | Supabase DB |
| **Evidence** | `app/api/me/active-workspace/route.ts` |

**Capabilities:**
- [x] View workspaces I belong to (inlined in GET /me)
- [x] Switch active workspace
- [ ] Leave workspace (BK-90 — Story pending)

---

### Miscellaneous

#### Feature: API Interactive Docs

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-DOC-001 |
| **Status** | Stable |
| **Endpoints** | `GET /api/openapi` (serves `public/openapi.json`) |
| **UI** | `/api/docs` (Scalar API Reference) |
| **Users** | Developers, QA |
| **Dependencies** | `@scalar/api-reference-react`, `@asteasolutions/zod-to-openapi` |
| **Evidence** | `app/api/docs/page.tsx`, `lib/openapi/registry.ts` |

**Capabilities:**
- [x] Auto-generated OpenAPI spec from Zod schemas
- [x] Interactive API reference UI

#### Feature: QA Guide

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-DOC-002 |
| **Status** | Stable (static page) |
| **Endpoints** | N/A |
| **UI** | `/qa-guide` |
| **Users** | QA engineers |
| **Evidence** | `app/(app)/qa-guide/page.tsx` |

**Capabilities:**
- [x] In-app QA documentation

#### Feature: Design Tokens Panel

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-DOC-003 |
| **Status** | Stable |
| **Endpoints** | N/A |
| **UI** | `/design-tokens` |
| **Users** | Developers, designers |
| **Evidence** | `app/design-tokens/page.tsx` |

**Capabilities:**
- [x] Visual panel of all design tokens

---

## 3. CRUD matrix

| Entity | Create | Read | Update | Delete | Evidence |
|--------|--------|------|--------|--------|----------|
| Workspace | ✅ | ✅ | ✅ | ❌ No UI | `/api/v1/workspaces` |
| WorkspaceMember | ⚠️ Via invite | ✅ | ⚠️ Role via invite | ❌ (leave not implemented) | `/api/v1/workspaces/[id]/invites` |
| Project | ✅ | ✅ | ❌ Not implemented | ❌ Not implemented | `/api/v1/workspaces/[id]/projects` |
| Module | ✅ | ✅ | ✅ Rename/move | ⚠️ Soft-delete | `/api/v1/projects/[id]/modules` |
| UserStory | ✅ | ✅ | ✅ | ⚠️ Soft-delete | `/api/v1/modules/[id]/user-stories` |
| AcceptanceCriterion | ✅ | ✅ | ✅ | ⚠️ Soft-delete | `/api/v1/user-stories/[id]/acceptance-criteria` |
| ATC | ✅ | ✅ (implied) | ✅ (version lock) | ❌ No delete endpoint | `/api/v1/atcs` |
| ATC Step | ✅ (embedded) | ✅ (embedded) | ✅ (full replace) | ✅ (embedded full replace) | `/api/v1/atcs` |
| ATC Assertion | ✅ (embedded) | ✅ (embedded) | ✅ (full replace) | ✅ (embedded full replace) | `/api/v1/atcs` |
| PersonalAccessToken | ✅ | ✅ | ❌ | ✅ Revoke (soft) | `/api/v1/tokens` |
| ImportJob | ✅ | ✅ | ❌ | ❌ | `/api/v1/imports` |
| WorkspaceInvite | ✅ | ✅ | ⚠️ Rotate (POST) | ✅ | `/api/v1/workspaces/[id]/invites` |

Legend: ✅ Full, ⚠️ Partial/conditional, ❌ Not available

---

## 4. API endpoint inventory

### Auth & Identity

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/` | API version discovery | Public |
| GET | `/api/v1/health` | Liveness probe | Public |
| GET | `/api/v1/me` | Current user + workspaces | Session or PAT |
| POST | `/api/v1/me/active-workspace` | Switch active workspace | Session or PAT |
| POST | `/api/v1/auth/magic-link` | Send magic-link OTP | Public |
| POST | `/api/v1/auth/signup` | Headless signup + PAT mint | Public |
| POST | `/api/v1/tokens` | Issue PAT | Session |
| GET | `/api/v1/tokens` | List tokens | Session |
| DELETE | `/api/v1/tokens/[id]` | Revoke token | Session |

### Workspaces

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/workspaces` | Create workspace | Authenticated |
| GET | `/api/v1/workspaces` | List workspaces | Authenticated |
| GET | `/api/v1/workspaces/[id]` | Workspace detail | Authenticated |
| PATCH | `/api/v1/workspaces/[id]` | Update workspace | Authenticated |
| POST | `/api/v1/workspaces/[id]/projects` | Create project | Authenticated |
| POST | `/api/v1/workspaces/[id]/invites` | Invite member | Admin |
| GET | `/api/v1/workspaces/[id]/invites` | List pending invites | Authenticated |
| POST | `/api/v1/workspaces/[id]/invites/[inviteId]` | Rotate invite | Admin |
| DELETE | `/api/v1/workspaces/[id]/invites/[inviteId]` | Revoke invite | Admin |
| POST | `/api/v1/invites/accept` | Accept invite | Public + token |

### Projects & Modules

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/projects/[id]/modules` | Create module | Authenticated |
| PATCH | `/api/v1/modules/[id]` | Rename/move/update module | Authenticated |
| DELETE | `/api/v1/modules/[id]` | Soft-delete module | Authenticated |

### User Stories & ACs

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/modules/[id]/user-stories` | Create user story | Authenticated |
| GET | `/api/v1/modules/[id]/user-stories` | List user stories | Authenticated |
| GET | `/api/v1/user-stories/[id]` | Get user story | Authenticated |
| PATCH | `/api/v1/user-stories/[id]` | Update user story | Authenticated |
| DELETE | `/api/v1/user-stories/[id]` | Soft-delete user story | Authenticated |
| POST | `/api/v1/user-stories/[id]/acceptance-criteria` | Create AC | Authenticated |
| GET | `/api/v1/user-stories/[id]/acceptance-criteria` | List ACs | Authenticated |
| PATCH | `/api/v1/acceptance-criteria/[id]` | Update AC | Authenticated |
| DELETE | `/api/v1/acceptance-criteria/[id]` | Soft-delete AC | Authenticated |

### ATCs

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/atcs` | Create ATC | Authenticated |
| PATCH | `/api/v1/atcs/[id]` | Edit ATC (full replace, optimistic lock) | Authenticated |

### Imports

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/imports` | Enqueue Jira import | Authenticated |
| GET | `/api/v1/imports` | List import jobs | Authenticated |
| GET | `/api/v1/imports/[id]` | Poll import job (SSE-ready) | Authenticated |

---

## 5. UI component inventory

### Pages

| Route | Component | Purpose | State |
|-------|-----------|---------|-------|
| `/login` | Login page (bilateral layout) | Email magic-link auth | Stable |
| `/auth/callback` | Route handler | Supabase callback | Stable |
| `/invites/accept` | Invite accept page | Accept workspace invite | Stable |
| `/onboarding` | Onboarding flow | First workspace setup | Stable |
| `/projects` | Project list page | Listing + navigation | Stable |
| `/projects/[slug]` | Project workbench | Module tree + ATC table | Stable |
| `/members` | Member management | Workspace member list | Stable |
| `/invites` | Invite list | Pending/inbound invites | Stable |
| `/qa-guide` | QA documentation page | Static guide | Stable |
| `/design-tokens` | Design tokens panel | Visual token reference | Stable |
| `/api/docs` | Scalar API reference | Interactive OpenAPI UI | Stable |

### Forms

| Component | Feature | State |
|-----------|---------|-------|
| `user-story-form.tsx` | Create/edit user story | Stable |
| Markdown editor | User story description | Stable (bugs BK-99, BK-100) |

### Modals / Dialogs

| Component | Feature | State |
|-----------|---------|-------|
| ATC builder | Create/edit ATC (assumed — shadcn dialog-based) | Stable |
| Invite dialog | Invite member (assumed) | Stable |
| Module create dialog | Create module (assumed) | Stable |

### App Shell Components

| Component | Location | Purpose |
|-----------|----------|---------|
| Sidebar | `app/(app)/layout.tsx` | Navigation (dark, surface-1) |
| Header | `app/(app)/layout.tsx` | App header + workspace switcher |
| Module tree | Project workbench | Nested module navigation |
| ATC table | Project workbench | Filterable ATC list |

### Design System Components (shadcn)

Button, Input, Select, Dialog, DropdownMenu, Tooltip, Tabs, Toast (sonner), Badge, Card, Table, Avatar, Separator.

---

## 6. Third-party integrations

| Service | Purpose | Package | Status | Features using it |
|---------|---------|---------|--------|-------------------|
| **Supabase** | Database + Auth + Storage | `@supabase/supabase-js`, `@supabase/ssr` | Active | All features |
| **Resend** | Transactional email (magic links, invites) | (SMTP/API) | Active | FEAT-AUTH-001, FEAT-WS-002 |
| **Jira** | Issue import (Epic → US) | `jira.js` | Beta | FEAT-IMP-001 |
| **Scalar** | API reference UI | `@scalar/api-reference-react` | Active | FEAT-DOC-001 |
| **Vercel** | Deployment + serverless functions | — | Active | All features (infra) |
| **GitHub OAuth** | Social login | Supabase Auth | Planned | FEAT-AUTH-003 |
| **Google OAuth** | Social login | Supabase Auth | Planned | FEAT-AUTH-003 |

---

## 7. Feature flags and WIP

### Flags

No feature-flag system discovered in code. `feature_flags` table exists in schema (migration 0009) but no runtime checks found. Env-based toggling not observed beyond `NEXT_PUBLIC_*` conventions.

### Planned features (from Jira PBIs + code TODOs)

| Feature | Evidence | Status |
|---------|----------|--------|
| OAuth sign-in (GitHub / Google) | BK-3 | Pending |
| Explorer views (tree/table/mind map) | BK-98 | Pending |
| Tab-based module navigation | BK-147 | Pending |
| ATC search & autocomplete | BK-20 | Pending |
| ATC propagation cascade | BK-21 | Pending |
| ATC usage report | BK-22 | Pending |
| ATC duplicate | BK-23 | Pending |
| Test builder (chain ATCs) | BK-27, BK-28 | Pending |
| Test view with expanded ATCs | BK-32 | Pending |
| Test tags (reserved + custom) | BK-33 | Pending |
| Manual run execution | BK-34–BK-39 | Pending |
| Defect filing + list + heatmap | BK-40–BK-43 | Pending |
| Traceability chain + coverage | BK-45–BK-50 | Pending |
| Activity feed | BK-49 | Pending |
| Export snapshots | BK-50 | Pending |
| Leave workspace | BK-90 | Pending |

---

## 8. QA relevance

### Feature test coverage matrix

| Feature ID | Unit | Integration | E2E | Status |
|------------|------|-------------|-----|--------|
| FEAT-AUTH-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-AUTH-002 | ❌ | ❌ | ❌ | No tests |
| FEAT-WS-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-WS-002 | ❌ | ❌ | ❌ | No tests |
| FEAT-PROJ-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-MOD-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-US-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-AC-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-ATC-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-IMP-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-MD-001 | ❌ | ❌ | ❌ | No tests |
| FEAT-DOC-001 | ❌ | ❌ | ❌ | No tests |

### High-risk features

| Feature | Risk | Reason |
|---------|------|--------|
| FEAT-AUTH-001 (Magic Link) | HIGH | Security — auth bypass locks out the entire system. Email delivery failures = no onboard. |
| FEAT-AUTH-002 (PAT) | HIGH | Security — scope enforcement incomplete (BK-117, BK-134). Token leak = workspace compromise. |
| FEAT-WS-002 (Invites) | CRITICAL | Security + data integrity — BK-60/61/62 bypass invite uniqueness and role enforcement. RLS gaps. |
| FEAT-IMP-001 (Jira Import) | MEDIUM | Data loss — async import with no retry. Staging credentials missing (BK-142). |
| FEAT-MD-001 (Markdown Editor) | LOW | Validation — upload limits not enforced (BK-99/100). Server-side bypass. |
| FEAT-MOD-001 (Module Move) | MEDIUM | Data integrity — BK-57: rename+move not atomic. Partial state possible. |

---

## 9. Discovery gaps

- **No automated tests exist** for any feature. Zero unit, integration, or E2E coverage. Assessment confirmed: maturity 0/4.
- **No CI/CD pipelines** — no GitHub Actions workflows in target. Build/lint checks run locally only.
- **OpenAPI spec is build-time generated** — no live spec served at runtime. `public/openapi.json` is force-static.
- **Feature flags table exists in schema** (`feature_flags`) but no runtime usage found in code paths. Likely reserved for future use.
- **Defect sync** (BK-43) mentions one-way external tracker sync — not yet implemented; target tracker not specified.
- **ATC endpoints lack GET/list** — no GET `/api/v1/atcs` found. Listing assumed via project workbench UI.
- **No GET /api/v1/projects** endpoint found — project listing assumed via workspace context UI.
- **No GET /api/v1/projects/[id]** detail endpoint — project routes are limited to module creation only.
- **SSO / OAuth** (BK-3) is planned but no code stubs found.
- **PAT scope enforcement** is implemented in the schema (scopes array) and minting but enforcement per-route is incomplete (BK-117, BK-134 confirm workspace:admin scope leak).
- **Activity feed** (BK-49) cites existing `activity_log` table — the data layer exists but no read-side API or UI is implemented.
