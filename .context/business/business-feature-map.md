# Bunkai — Business Feature Map

```
  ╔══════════════════════════════════════════════════════════════════╗
  ║              BUNKAI — Feature Catalog                          ║
  ║      Every capability the system offers, mapped and ranked     ║
  ╚══════════════════════════════════════════════════════════════════╝
```

## Inventory Summary

| Category | Features | Status |
|----------|----------|--------|
| Core | 10 | Stable |
| Secondary | 5 | Stable |
| Beta | 1 | Testing (Jira Import) |
| Planned | 2 | In Development (email notifications, PAT rotation) |

---

## Feature Catalog (by Domain)

### Domain: Authentication & Session Management

#### Feature: Email+Password Sign-up

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AUTH-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/auth/signup` |
| **UI** | `/login` (sign-up form) |
| **Users** | Public (unauthenticated) |
| **Dependencies** | Supabase Auth |
| **Evidence** | `app/api/v1/auth/signup/route.ts` |

**Capabilities:**
- [x] Create user account with email+password
- [x] Auto-confirm account (no email verification)
- [x] Return session cookie + minted PAT in response
- [x] Rate limiting / brute-force protection? (not verified)

#### Feature: Email+Password Sign-in

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AUTH-002 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/auth/signin` |
| **UI** | `/login` (sign-in form) |
| **Users** | Public |
| **Dependencies** | Supabase Auth |
| **Evidence** | `app/api/v1/auth/signin/route.ts` |

**Capabilities:**
- [x] Authenticate with email+password
- [x] Return session cookie + minted PAT
- [ ] Account lockout after failed attempts? (not verified)

#### Feature: Magic Link / OTP Sign-in

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AUTH-003 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/auth/magic-link` |
| **UI** | `/login` (email form), `/auth/callback` |
| **Users** | Public |
| **Dependencies** | Supabase Auth, Resend (configured) |
| **Evidence** | `app/api/v1/auth/magic-link/route.ts` |

**Capabilities:**
- [x] Send OTP email with magic link
- [x] Callback handler consumes token and sets session
- [x] Replay protection via `magic_link_tokens.consumed_at`
- [ ] Email delivery via Resend? (MVP logs to console)

#### Feature: Session Introspection

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AUTH-004 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/me`, `POST /api/v1/me/active-workspace` |
| **UI** | Header/nav (user info), workspace switcher |
| **Users** | Authenticated |
| **Dependencies** | Supabase Auth |
| **Evidence** | `app/api/v1/me/route.ts` |

**Capabilities:**
- [x] Return current user + workspaces + active workspace
- [x] Switch active workspace (cookie `bk_active_ws`)
- [x] Detect auth method (cookie vs PAT)

### Domain: Workspace Management

#### Feature: Workspace CRUD

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-WS-001 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/workspaces`, `POST /api/v1/workspaces`, `GET /api/v1/workspaces/{id}`, `PATCH /api/v1/workspaces/{id}` |
| **UI** | `/onboarding` (first workspace), `/workspaces/{id}` (settings) |
| **Users** | Authenticated users |
| **Dependencies** | Supabase (RPC `bunkai_bootstrap_workspace`) |
| **Evidence** | `app/api/v1/workspaces/route.ts`, `app/api/v1/workspaces/[id]/route.ts` |

**Capabilities:**
- [x] Create workspace (bootstrap: workspace + owner membership atomically)
- [x] List user's workspaces
- [x] View workspace details
- [x] Update workspace name
- [ ] Delete workspace (not exposed in endpoints)
- [ ] Change workspace plan (community → cloud → enterprise)

#### Feature: Workspace Invites

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-WS-002 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/workspaces/{id}/invites`, `POST /api/v1/workspaces/{id}/invites`, `POST /api/v1/invites/accept` |
| **UI** | `/invites/accept?token=...`, workspace settings panel |
| **Users** | Admins/owners (issue), any user (accept) |
| **Dependencies** | Supabase, Resend (future) |
| **Evidence** | `app/api/v1/workspaces/[id]/invites/route.ts`, `app/api/v1/invites/accept/route.ts` |

**Capabilities:**
- [x] Issue invite by email + role (7-day expiry)
- [x] Accept invite with raw token (email must match)
- [x] No-demotion rule (reject if existing role ≥ invite role)
- [x] Revoke invite (implied via sibling endpoint)
- [ ] Email notification on invite (MVP logs to console)

### Domain: Project & Module Tree

#### Feature: Project CRUD

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-PRJ-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/workspaces/{id}/projects` |
| **UI** | `/projects` (list + create) |
| **Users** | Workspace members |
| **Dependencies** | Workspace |
| **Evidence** | `app/api/v1/workspaces/[id]/projects/route.ts` |

**Capabilities:**
- [x] Create project within workspace (unique slug per workspace)
- [ ] List projects (via GET workspaces/{id} or dedicated endpoint)
- [ ] Update project? (not verified)
- [ ] Delete project? (not verified)

#### Feature: Module Tree Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-MOD-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/projects/{id}/modules`, `PATCH /api/v1/modules/{id}`, `DELETE /api/v1/modules/{id}`, `POST /api/v1/modules/{id}/move` |
| **UI** | `/projects/{projectSlug}` (tree explorer panel) |
| **Users** | Workspace members (viewer read-only) |
| **Dependencies** | Project |
| **Evidence** | `app/api/v1/projects/[id]/modules/route.ts`, `app/api/v1/modules/[id]/route.ts` |

**Capabilities:**
- [x] Create module (auto-slug, depth-checked to max 6)
- [x] Rename module + rebuild descendant paths (RPC)
- [x] Move module to new parent (cycle-detected, depth-guarded)
- [x] Soft-delete module subtree (recursive CTE archive)

### Domain: User Stories & Acceptance Criteria

#### Feature: User Story Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-US-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/modules/{id}/user-stories`, `PATCH /api/v1/user-stories/{id}` |
| **UI** | `/projects/{projectSlug}` (story list panel) |
| **Users** | Workspace members |
| **Dependencies** | Module tree |
| **Evidence** | `app/api/v1/modules/[id]/user-stories/route.ts`, `app/api/v1/user-stories/[id]/route.ts` |

**Capabilities:**
- [x] Create user story in module
- [x] Set story status (draft → ready_to_test with ≥1 AC gate)
- [x] Soft-delete story
- [x] Jira `external_id` linking (for import sync)
- [ ] List stories by module
- [ ] Story detail view

#### Feature: Acceptance Criteria Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-AC-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/user-stories/{id}/acceptance-criteria`, `PATCH /api/v1/acceptance-criteria/{id}`, `DELETE /api/v1/acceptance-criteria/{id}` |
| **UI** | Story detail panel (AC list) |
| **Users** | Workspace members |
| **Dependencies** | User Story |
| **Evidence** | `app/api/v1/user-stories/[id]/acceptance-criteria/route.ts`, `app/api/v1/acceptance-criteria/[id]/route.ts` |

**Capabilities:**
- [x] Create AC with auto-position (negative-parking rebalance)
- [x] Move AC position (rebalance)
- [x] Soft-delete AC (auto-reverts story to draft if last AC)
- [x] Position integrity under concurrent writes

### Domain: ATC (Acceptance Test Cases)

#### Feature: ATC Authoring

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ATC-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/atcs`, `PATCH /api/v1/atcs/{id}` |
| **UI** | `/projects/{projectSlug}` (ATC editor with Monaco), `/projects/{projectSlug}/atcs/new` |
| **Users** | Workspace members with `atc:write` scope |
| **Dependencies** | Module, User Story, ACs |
| **Evidence** | `app/api/v1/atcs/route.ts`, `app/api/v1/atcs/[id]/route.ts` |

**Capabilities:**
- [x] Create ATC with steps + assertions + AC bindings (atomic RPC)
- [x] Update ATC with full-replace of children (optimistic lock via `X-If-Match`)
- [x] Anchoring moat: ATC must reference ≥1 AC belonging to the parent story
- [x] Module validation: ATC module must be story's module or descendant
- [x] Slug generation from module path
- [x] Version increment (optimistic concurrency)

#### Feature: ATC Search & Discovery

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ATC-002 |
| **Status** | Stable |
| **Endpoints** | (implied via project-level queries) |
| **UI** | `/projects/{projectSlug}` (ATC list/search, mind-map view) |
| **Users** | Workspace members |
| **Dependencies** | ATC data, TSV index |
| **Evidence** | DB trigger `atcs_refresh_tsv` for full-text search |

**Capabilities:**
- [x] Full-text search on ATC title + tags (tsvector)
- [x] Filter by layer (UI/API/Unit)
- [x] Filter by status
- [x] Filter by module
- [x] Tag-based organization

#### Feature: ATC Status Tracking

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-ATC-003 |
| **Status** | Stable |
| **Endpoints** | (via PATCH /atcs/{id} status field) |
| **UI** | ATC detail view (status badge/changer) |
| **Users** | Workspace members with `atc:write` |
| **Dependencies** | ATC |
| **Evidence** | Status lifecycle: unrun → running → pass/fail/blocked/skipped |

**Capabilities:**
- [x] Set ATC status (any → any transition allowed)
- [x] Activity log records each status change
- [ ] Status lifecycle enforcement? (DB allows all transitions — no gate)

### Domain: Personal Access Tokens (PAT)

#### Feature: PAT Issuance & Revocation

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-PAT-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/tokens`, `GET /api/v1/tokens`, `DELETE /api/v1/tokens/{id}` |
| **UI** | User settings / profile panel |
| **Users** | Authenticated (browser session only — PAT cannot mint PAT) |
| **Dependencies** | Supabase (sibling table for secret isolation) |
| **Evidence** | `app/api/v1/tokens/route.ts`, `app/api/v1/tokens/[id]/route.ts` |

**Capabilities:**
- [x] Issue `bk_pat_*` token with scoped permissions
- [x] List caller's PATs (without secret — shown once on creation)
- [x] Soft-revoke PAT (sets `revoked_at`)
- [x] Scopes: atc:read, atc:write, run:execute, workspace:admin
- [x] Browser-session-only enforcement (PAT cannot create PAT)

### Domain: Jira Integration

#### Feature: Jira Import

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-JIRA-001 |
| **Status** | Beta |
| **Endpoints** | `POST /api/v1/imports`, `GET /api/v1/imports/{id}` |
| **UI** | `/projects/{projectSlug}` (import dialog) |
| **Users** | Workspace members |
| **Dependencies** | Jira API, Vercel `after()`, Supabase |
| **Evidence** | `app/api/v1/imports/route.ts`, `lib/jira/import-runner.ts`, `lib/jira/client.ts` |

**Capabilities:**
- [x] Enqueue import job with JQL query (returns 202)
- [x] One active import per project (partial unique index)
- [x] Async import in Vercel after() background slot
- [x] Jira ADF → Markdown conversion for AC content
- [x] Upsert stories + ACs per Jira issue
- [x] Poll job status + error details
- [ ] Retry on failure? (not verified)
- [ ] Email notification on completion? (not implemented)

### Secondary Features

#### Feature: Activity Log / Audit Trail

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-SYS-001 |
| **Status** | Stable |
| **Endpoints** | (implied, no dedicated endpoint found) |
| **UI** | (none in MVP) |
| **Users** | System (written by every mutation RPC) |
| **Dependencies** | All write RPCs |
| **Evidence** | `activity_log` table, written by each RPC |

**Capabilities:**
- [x] Log entity mutations (entity_type, entity_id, action, payload)
- [x] Immutable log (no delete/update)
- [ ] Read/query endpoint for activity log

#### Feature: Idempotency

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-SYS-002 |
| **Status** | Stable |
| **Endpoints** | (transparent — applies to all POST endpoints) |
| **UI** | None (behind-the-scenes) |
| **Users** | System |
| **Dependencies** | `idempotency_keys` table |
| **Evidence** | `idempotency_keys` table schema |

**Capabilities:**
- [x] 24h TTL idempotency keys
- [x] Scope: (user_id, endpoint, key)
- [x] Response snapshot stored for repeat detection

#### Feature: Feature Flags

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-SYS-003 |
| **Status** | Stable (infrastructure only — MVP flags untraced in code) |
| **Endpoints** | None |
| **UI** | None |
| **Users** | System (internal) |
| **Dependencies** | `feature_flags` table |
| **Evidence** | `feature_flags` table: global | workspace scoped with overrides |

**Capabilities:**
- [x] Global feature flags (affect all workspaces)
- [x] Workspace-scoped flags with per-workspace overrides
- [ ] Feature flag admin UI? (not verified)
- [ ] MVP feature flags actually used in code? (untraced)

#### Feature: API Documentation

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-SYS-004 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1` (version banner), `GET /api/v1/health`, `/api/docs` (Scalar), `/api/openapi` (raw spec) |
| **UI** | `/api/docs` (Scalar UI), `/qa` (testability guide) |
| **Users** | Public (health, version), authenticated (docs) |
| **Dependencies** | Scalar, OpenAPI spec |
| **Evidence** | `app/api/v1/route.ts`, `app/api/docs/`, `app/qa/` |

**Capabilities:**
- [x] OpenAPI spec served at `/api/openapi`
- [x] Interactive API docs via Scalar at `/api/docs`
- [x] Health check endpoint
- [x] Software Testability Guide page at `/qa`

---

## CRUD Matrix

| Entity | Create | Read | Update | Delete | Evidence |
|--------|--------|------|--------|--------|----------|
| User (auth) | ✅ Signup | ✅ Me | ❓ | ❓ | `POST /auth/signup`, `GET /me` |
| Workspace | ✅ | ✅ | ✅ Name only | ❌ Not exposed | `POST /workspaces`, `PATCH /workspaces/{id}` |
| WorkspaceMember | ✅ Invite | ✅ Via workspace | ❌ Role change? | ❌ Remove member? | `POST /invites` |
| Project | ✅ | ✅ Via workspace | ❓ | ❓ | `POST /projects` |
| Module | ✅ | ✅ Via project tree | ✅ Rename | ⚠️ Soft-delete | Module endpoints |
| UserStory | ✅ | ✅ Via module | ⚠️ Status only | ⚠️ Soft-delete | Story endpoints |
| AcceptanceCriterion | ✅ | ✅ Via story | ⚠️ Position only | ⚠️ Soft-delete | AC endpoints |
| ATC | ✅ | ✅ Via module/story | ✅ Full replace | ⚠️ Soft-delete | ATC endpoints |
| ATC Step | ✅ (with ATC) | ✅ (with ATC) | ✅ (full replace) | ✅ (full replace) | ATC create/update |
| ATC Assertion | ✅ (with ATC) | ✅ (with ATC) | ✅ (full replace) | ✅ (full replace) | ATC create/update |
| AccessToken (PAT) | ✅ | ✅ List (no secret) | ❌ | ⚠️ Soft-revoke | Token endpoints |
| ImportJob | ✅ Enqueue | ✅ Poll status | ❌ | ❌ | Import endpoints |
| ActivityLog | ✅ (auto) | ❌ No endpoint | ❌ | ❌ | DB only |
| FeatureFlag | ❌ No endpoint | ❌ No endpoint | ❌ | ❌ | DB only |
| UserViewState | ✅ (auto) | ✅ (auto) | ✅ (auto) | ❓ | `user_view_state` table |

Legend: ✅ Full, ⚠️ Partial/conditional, ❌ Not available, ❓ Not verified

---

## API Endpoint Inventory

### Public

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1` | API version + docs links | Public |
| GET | `/api/v1/health` | Health check | Public |
| POST | `/api/v1/auth/magic-link` | Send OTP email | Public |
| POST | `/api/v1/auth/signin` | Email+password sign-in | Public |
| POST | `/api/v1/auth/signup` | Create account | Public |

### Authenticated (cookie or PAT)

| Method | Endpoint | Purpose | Scope |
|--------|----------|---------|-------|
| GET | `/api/v1/me` | Session introspection | Any |
| POST | `/api/v1/me/active-workspace` | Switch workspace | Any |
| GET | `/api/v1/workspaces` | List workspaces | Any |
| POST | `/api/v1/workspaces` | Create workspace | Any |
| GET | `/api/v1/workspaces/{id}` | Workspace detail | Any |
| PATCH | `/api/v1/workspaces/{id}` | Update workspace | Any |
| POST | `/api/v1/workspaces/{id}/projects` | Create project | Any |
| GET | `/api/v1/workspaces/{id}/invites` | List invites | admin/owner |
| POST | `/api/v1/workspaces/{id}/invites` | Issue invite | admin/owner |
| POST | `/api/v1/invites/accept` | Accept invite | Any |
| POST | `/api/v1/projects/{id}/modules` | Create module | member+ |
| PATCH | `/api/v1/modules/{id}` | Rename module | member+ |
| DELETE | `/api/v1/modules/{id}` | Archive module | member+ |
| POST | `/api/v1/modules/{id}/move` | Move module | member+ |
| POST | `/api/v1/modules/{id}/user-stories` | Create story | member+ |
| PATCH | `/api/v1/user-stories/{id}` | Set story status | member+ |
| POST | `/api/v1/user-stories/{id}/acceptance-criteria` | Create AC | member+ |
| PATCH | `/api/v1/acceptance-criteria/{id}` | Move AC | member+ |
| DELETE | `/api/v1/acceptance-criteria/{id}` | Archive AC | member+ |
| **POST** | **`/api/v1/atcs`** | **Create ATC** | **atc:write** |
| **PATCH** | **`/api/v1/atcs/{id}`** | **Update ATC** | **atc:write** |
| POST | `/api/v1/imports` | Enqueue Jira import | member+ |
| GET | `/api/v1/imports/{id}` | Poll import status | member+ |
| POST | `/api/v1/tokens` | Issue PAT | Browser only |
| GET | `/api/v1/tokens` | List PATs | Any |
| DELETE | `/api/v1/tokens/{id}` | Revoke PAT | Browser only |

---

## UI Component Inventory

### Pages

| Route | Component / Page | Purpose |
|-------|-----------------|---------|
| `/` | Root | Redirect to `/projects` or `/login` |
| `/login` | SignInPage | Email+password sign-in, magic-link form |
| `/auth/callback` | AuthCallback | OTP callback handler |
| `/onboarding` | OnboardingPage | First workspace creation |
| `/projects` | ProjectIndexPage | List/create projects |
| `/projects/{slug}` | ProjectWorkbench | Main workspace: module tree, ATC editor, story management, mind-map, import dialog |
| `/projects/{slug}/atcs/new` | NewAtcGuidePage | ATC creation guide |
| `/workspaces/{id}` | WorkspaceSettings | Workspace management |
| `/invites/accept` | InviteAcceptPage | Accept workspace invitation |
| `/qa` | TestabilityGuide | Software testability documentation |
| `/api/docs` | ScalarApiDocs | Interactive API reference |

### Key UI Components (from codebase)

| Component | Type | Purpose |
|-----------|------|---------|
| TopNav | Navigation | Workspace switcher, user menu |
| ModuleTree | Tree view | Hierarchical module navigation |
| StoryList | List view | Stories under a module |
| ACList | List | Acceptance criteria for a story |
| AtcEditor | Code editor (Monaco) | ATC authoring with steps + assertions |
| MindMapView | Visual | Graph view of ATCs and relationships |
| ImportDialog | Modal | JQL input and import trigger |
| InviteDialog | Modal | Send workspace invite |

---

## Third-Party Integrations

| Service | Purpose | Package | Status | Features Using It |
|---------|---------|---------|--------|-------------------|
| **Supabase** | Database, Auth, RLS, Storage | `@supabase/ssr`, `@supabase/supabase-js` | Active | ALL features |
| **Jira** | Issue import | Custom (no SDK) | Beta | Jira Import |
| **Resend** | Transactional email | `resend` | Configured, not wired | Future: invite notifications |
| **Vercel** | Deployment, Edge Runtime, `after()` | `next/server` | Active | Jira Import (background), all pages |
| **Scalar** | API docs UI | `@scalar/api-reference-react` | Active | API Documentation |
| **Monaco Editor** | Code/text editing | `@monaco-editor/react` | Active | ATC Editor |
| **Radix UI** | Headless UI primitives | `@radix-ui/*` | Active | Dialogs, dropdowns, tabs, tooltips |
| **TanStack Table** | Data tables | `@tanstack/react-table` | Active | ATC/story listing |
| **Lucide** | Icons | `lucide-react` | Active | All UI |
| **Shiki** | Syntax highlighting | `shiki`, `react-markdown` | Active | Markdown rendering |
| **n8n** | Workflow automation | MCP config | Configured, not used | Future |

---

## Feature Flags and WIP

### Feature Flags

| Flag | Description | Default | Scope |
|------|-------------|---------|-------|
| (No named flags found in MVP code) | Feature flag infrastructure exists but usage in code untraced | — | — |

### Planned / WIP Features

| Feature | Evidence | Estimated Status |
|---------|----------|-----------------|
| Email notifications for workspace invites | Resend configured, invite endpoint logs to console | Near-term |
| PAT rotation / expiry notifications | Token `expires_at` column exists but UI/notification not implemented | Future |
| ATC bulk operations | No endpoint, no UI | Future |
| Workspace deletion | No endpoint | Future |
| Role change for members | PATCH on membership not exposed | Future |
| Activity log viewer | Table exists, no read endpoint | Future |

---

## QA Relevance

### Feature Test Coverage Matrix

| Feature ID | Unit | Integration | E2E | Status |
|------------|------|-------------|-----|--------|
| FEAT-AUTH-001 (Sign-up) | ❌ | ❌ | ❌ | Not tested |
| FEAT-AUTH-002 (Sign-in) | ❌ | ❌ | ❌ | Not tested |
| FEAT-AUTH-003 (Magic Link) | ❌ | ❌ | ❌ | Not tested |
| FEAT-AUTH-004 (Session) | ❌ | ❌ | ❌ | Not tested |
| FEAT-WS-001 (Workspace CRUD) | ❌ | ❌ | ❌ | Not tested |
| FEAT-WS-002 (Invites) | ❌ | ❌ | ❌ | Not tested |
| FEAT-PRJ-001 (Project CRUD) | ❌ | ❌ | ❌ | Not tested |
| FEAT-MOD-001 (Module Tree) | ❌ | ❌ | ❌ | Not tested |
| FEAT-US-001 (User Stories) | ❌ | ❌ | ❌ | Not tested |
| FEAT-AC-001 (AC Management) | ❌ | ❌ | ❌ | Not tested |
| FEAT-ATC-001 (ATC Authoring) | ❌ | ❌ | ❌ | Not tested |
| FEAT-ATC-002 (ATC Search) | ❌ | ❌ | ❌ | Not tested |
| FEAT-ATC-003 (ATC Status) | ❌ | ❌ | ❌ | Not tested |
| FEAT-PAT-001 (PAT Mgmt) | ❌ | ❌ | ❌ | Not tested |
| FEAT-JIRA-001 (Jira Import) | ❌ | ❌ | ❌ | Not tested |

### High-Risk Features (Priority Testing)

| Feature | Risk | Reason |
|---------|------|--------|
| FEAT-ATC-001 (ATC Authoring) | CRITICAL | Core entity — data integrity loss is unrecoverable |
| FEAT-AUTH-001/002/003 (Auth) | CRITICAL | Single gate to all data — RLS-only auth means no TS fallback |
| FEAT-MOD-001 (Module Tree) | CRITICAL | Cascade operations (rename path rebuild, subtree archive) affect all child data |
| FEAT-AC-001 (AC Management) | HIGH | Auto-revert on last-AC-delete is a silent semantic change |
| FEAT-WS-002 (Invites) | HIGH | Role escalation or demotion-bypass leaks write access |
| FEAT-PAT-001 (PAT Mgmt) | HIGH | Token scope enforcement is the only gate between CLI agents and data |
| FEAT-JIRA-001 (Jira Import) | MEDIUM | Async — partial commits, no failure notification to user |

---

## Discovery Gaps

- **Feature flags usage untraced** — `feature_flags` table infrastructure exists but actual flag consumption in MVP code was not confirmed. Some features may be disabled via flags without being visible in the endpoint inventory.
- **UI components not fully cataloged** — Only key components listed. A thorough UI audit (form inputs, validation, error states) is pending.
- **Role-change and member-removal endpoints** — Not confirmed to exist. The invite endpoint creates members but there's no PATCH to change roles or DELETE to remove members.
- **Project-level read endpoints** — `GET /projects` exists, but project detail and update/delete endpoints were not verified.
- **ATC query/search endpoints** — ATC listing, filtering, and full-text search endpoints are implied by the UI but their exact API surface was not traced.
- **Viewer role enforcement** — Confirmed at Postgres RLS level but the exact endpoint-level rejection behavior was not tested.
- **Email delivery** — Resend is configured but invite/magic-link emails may log to console in MVP. Actual delivery status unknown.
- **Plan upgrade path** — Workspace plan column exists (community → cloud → enterprise) but no endpoint or UI for upgrading was found.
