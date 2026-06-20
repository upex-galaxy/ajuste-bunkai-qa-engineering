# Business API Map — Bunkai (upex-bunkai-tms)

> Generated: 2026-06-20 | Last verified against OpenAPI on 2026-06-20
> Sources: OpenAPI spec (`public/openapi.json`), target repo code scan, Jira PBIs, SRS, feature-map (FEAT-*)

---

## 1. Executive summary

Bunkai's API lets QA engineers and CI/CD agents decompose user stories into executable ATCs (Acceptance Test Cases), organize them into a module hierarchy, chain them into tests, execute runs, and track coverage — all through a versioned `/api/v1` surface.

The API operates in dual auth mode: browser users authenticate via Supabase SSR cookies (magic-link), while CLI/agent callers use `bk_pat_*` bearer tokens. Every response follows a canonical envelope `{ success, data, error }`, and mutation endpoints honor idempotency keys for safe retries.

Authorization is workspace-scoped RLS enforced at the database level. Four roles (viewer → member → admin → owner) gate read/write/admin operations per workspace, with the `bunkai_user_role()` helper resolving the caller's privileges from either cookie session or PAT identity.

---

## 2. Permission & auth model

### Tier table

| Tier | Who it applies to | How to acquire | Where enforced |
|------|-------------------|----------------|----------------|
| **Public** | Unauthenticated visitors | None | `/api/v1/`, `/api/v1/health`, route-level public check |
| **Session (cookie)** | Browser users | Magic-link → Supabase SSR cookie (`sb-*-auth-token`) | `middleware.ts` (route matcher), `lib/api/principal.ts` via `resolveIdentity()` |
| **Bearer (PAT)** | CLI / CI / AI agents | `POST /api/v1/tokens` → `bk_pat_<12-prefix>.<base64url-secret>` | `middleware/bearer.ts` (verify bearer token), SHA-256 hash lookup in `personal_access_tokens` |
| **Workspace member** | Authenticated users | Invite + accept or create workspace | RLS policies via `bunkai_is_workspace_member()` (SECURITY DEFINER) |
| **Workspace admin** | Users with admin/owner role | Role assignment on invite/create | RLS via `bunkai_is_workspace_admin()`, app-level checks |
| **Scope-gated (PAT)** | PAT callers | Scope array on token (`atc:read`, `atc:write`, `run:execute`, `workspace:admin`) | Incomplete enforcement (BK-117, BK-134: `workspace:admin` leak to members) |

### Token flow — Cookie auth (browser)

```
User ──► POST /api/v1/auth/magic-link { email }
          │
          ▼
        Supabase sends OTP email via Resend
          │
          ▼
        User clicks link ──► GET /auth/callback ──► Supabase SSR sets cookie
          │
          ▼
        Browser ──► GET /projects (cookie present) ──► middleware.ts refreshes session
          │                                                  │
          ▼                                                  ▼
        Route handler                           `resolveIdentity()` → `db` client (RLS-scoped)
```

### Token flow — Bearer auth (PAT)

```
CLI ──► POST /api/v1/auth/signup { email, password } ──► Gets session + PAT in one response
  │
  ├── Subsequent calls: Authorization: Bearer bk_pat_<prefix>.<secret>
  │
  ▼
resolveIdentity() ──► No cookie ──► verifyBearerToken() ──► SHA-256(secret) lookup
  │
  ▼
personal_access_tokens (check: not expired, not revoked)
  │
  ▼
db client attached with RLS-scoped identity
```

---

## 3. Critical business journeys

### Journey 1: QA engineer signs up and creates a workspace

**Business purpose**: A new user (QA engineer, developer, or test manager) provisions an account and a workspace in under 30 seconds.

```
Browser                    API                             Supabase            DB
  │                         │                                │                 │
  ├─ POST /auth/magic-link ─┤                                │                 │
  │                         ├───────────────────────────────►│ signInWithOtp() │
  │                         │                                │                 │
  │     (email with OTP)    │                                │                 │
  │                         │                                │                 │
  ├─ GET /auth/callback ────┤                                │                 │
  │                         ├───────────────────────────────►│ verifyOtp()     │
  │                         │◄───────────────────────────────┤ session cookie  │
  │                         │                                │                 │
  ├─ POST /api/v1/workspaces ┤                                │                 │
  │                         ├─ resolveIdentity() ───────────►│                 │
  │                         │                                ├─ RLS check ────►│ workspace_members
  │                         │                                │                 │  (auto created by
  │                         │                                │                 │   bootstrap trigger)
  │◄────────────────────────┤ workspace created              │                 │
  │                         │                                │                 │
  └─ Redirect to /projects ─┘                                │                 │
```

**Endpoints involved**: `POST /auth/magic-link`, `GET /auth/callback` (Supabase), `POST /api/v1/workspaces`
**Entities touched**: `users` (Supabase Auth), `workspaces`, `workspace_members`
**Feature IDs**: FEAT-AUTH-001, FEAT-WS-001

**Why it matters**: If this breaks, nobody can onboard — the entire system is gated behind workspace membership.

---

### Journey 2: QA lead invites a teammate with a role

**Business purpose**: A QA lead adds a new team member to the workspace with the correct permission level.

```
Admin                     API                         Supabase              DB
  │                        │                            │                    │
  ├─ POST /workspaces/W/   │                            │                    │
  │   invites { email,     │                            │                    │
  │   role: "member" }     │                            │                    │
  │                        ├─ resolveIdentity() ───────►│                    │
  │                        │  → is_admin check          │                    │
  │                        │                            ├─ RLS check ───────►│ workspace_members
  │                        │                            │                    │
  │                        ├─ Validate uniqueness ─────►│                    │
  │                        │  (BROKEN: BK-60, BK-61)    │                    │
  │                        │                            │                    │
  │                        ├─ Generate token + hash ───►│                    │
  │                        │                            ├─ INSERT ──────────►│ workspace_invites
  │                        │                            │                    │
  │◄───────────────────────┤ { invite, token,           │                    │
  │                        │   accept_url }             │                    │
  │                        │                            │                    │
  │  (share accept_url     │                            │                    │
  │   with teammate)       │                            │                    │
```

**Endpoints involved**: `POST /workspaces/[id]/invites`, `POST /api/v1/invites/accept`
**Entities touched**: `workspace_invites`, `workspace_members`
**Feature IDs**: FEAT-WS-002

**Why it matters**: Known bugs BK-60 (no uniqueness check against active members), BK-61 (duplicate invites), BK-62 (role overwrite via upsert) mean invite integrity does not hold — the most critical integrity gap in the system.

---

### Journey 3: QA engineer structures a project with nested modules

**Business purpose**: An engineer decomposes a feature area into a hierarchy of modules (max depth 6) to organize user stories and ATCs.

```
User                      API                                   DB
  │                        │                                     │
  ├─ POST /workspaces/W/   │                                     │
  │   projects { name }    │                                     │
  │                        ├─ resolveIdentity()                  │
  │                        ├─ Auto-derive slug from name         │
  │                        ├─ INSERT ───────────────────────────►│ projects
  │◄───────────────────────┤ { project }                         │
  │                        │                                     │
  ├─ POST /projects/P/     │                                     │
  │   modules              │                                     │
  │   { name,              │                                     │
  │    parent_module_id }  │                                     │
  │                        ├─ INSERT (materialized path) ───────►│ modules
  │◄───────────────────────┤ { module, warning? }                │
  │                        │                                     │
  ├─ PATCH /modules/M      │                                     │
  │   { parent_module_id:  │                                     │
  │   null }               │                                     │
  │                        ├─ move_module() RPC ────────────────►│ modules (path recompute)
  │◄───────────────────────┤ { module }                          │
  │                        │                                     │
  └─ DELETE /modules/M ────┤                                     │
                           ├─ soft_delete_module_cascade() ─────►│ modules + children
                           │                                     │  (archived_at set)
```

**Endpoints involved**: `POST /workspaces/[id]/projects`, `POST /projects/[id]/modules`, `PATCH /modules/[id]`, `DELETE /modules/[id]`
**Entities touched**: `projects`, `modules`
**Feature IDs**: FEAT-PROJ-001, FEAT-MOD-001

**Why it matters**: BK-57 (rename+move not atomic) and BK-67 (toast suppressed at depth >=5) make the module tree fragile. Re-parenting with path recompute is the most complex DB operation in the system.

---

### Journey 4: QA engineer authors ATCs for a user story

**Business purpose**: An engineer creates ATCs (Acceptance Test Cases) under a user story, linking them to acceptance criteria, to build the executable test library.

```
User                      API                                     DB
  │                        │                                       │
  ├─ POST /modules/M/     │                                       │
  │   user-stories        │                                       │
  │   { title,            │                                       │
  │    external_id:       │                                       │
  │    "BK-14" }          │                                       │
  │                        ├─ INSERT ────────────────────────────►│ user_stories
  │◄───────────────────────┤ { user_story }                       │
  │                        │                                       │
  ├─ POST /user-stories/US│                                       │
  │   /acceptance-criteria │                                       │
  │   { title }            │                                       │
  │                        ├─ INSERT ────────────────────────────►│ acceptance_criteria
  │                        │  (position auto-assigned)            │  (partial unique index)
  │◄───────────────────────┤ { ac }                                │
  │                        │                                       │
  │  (repeat for each AC)  │                                       │
  │                        │                                       │
  ├─ POST /api/v1/atcs ────┤                                       │
  │   { title,             │                                       │
  │     module_id,         │                                       │
  │     user_story_id,     │                                       │
  │     acceptance_crite-  │                                       │
  │     rion_ids: [...],   │                                       │
  │     layer: "API",      │                                       │
  │     steps: [...],      │                                       │
  │     assertions: [...]  │                                       │
  │   }                    │                                       │
  │                        ├─ create_atc_v1() RPC ───────────────►│ atcs + atc_steps
  │                        │  (SECURITY DEFINER,                   │  + atc_assertions
  │                        │   implicit actor_id from PAT)         │  + atc_acceptance_criteria
  │◄───────────────────────┤ { atc }                                │
```

**Endpoints involved**: `POST /modules/[id]/user-stories`, `POST /user-stories/[id]/acceptance-criteria`, `POST /api/v1/atcs`, `PATCH /api/v1/atcs/[id]`
**Entities touched**: `user_stories`, `acceptance_criteria`, `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`
**Feature IDs**: FEAT-US-001, FEAT-AC-001, FEAT-ATC-001

**Why it matters**: This is the **core authoring flow** — the system's primary value proposition. BK-96 (PATCH returns 412 instead of 200) means edit feedback is broken. ATCs use optimistic locking (`version` column) — stale writes return 409.

---

### Journey 5: Jira import syncs epics into user stories

**Business purpose**: A team migrates their existing Jira backlog into Bunkai, linking each issue to a module and preserving cross-references.

```
User                      API                          jira.js            DB
  │                        │                             │                 │
  ├─ POST /api/v1/imports  │                             │                 │
  │   { project_id,        │                             │                 │
  │     jql: "project=BK"  │                             │                 │
  │   }                    │                             │                 │
  │                        ├─ resolveIdentity()          │                 │
  │                        ├─ Queue check (BK-142) ─────┤                 │
  │                        │  (only 1 active per         │                 │
  │                        │   project)                  │                 │
  │                        ├─ INSERT ───────────────────┤                 ├─► import_jobs
  │◄───────────────────────┤ { import_job, status:       │                 │
  │                        │   "queued" }                │                 │
  │                        │                             │                 │
  │  (async)               │                             │                 │
  │                        ├─ Fetch issues via jira.js ──►│ Jira REST API  │
  │                        │◄────────────────────────────┤ issues          │
  │                        │                             │                 │
  │                        ├─ For each issue:            │                 │
  │                        │  ├─ Upsert user_story ──────┤                 ├─► user_stories
  │                        │  │  (external_id unique)    │                 │
  │                        │  ├─ Map component → module  │                 │
  │                        │  └─ Log errors per issue    │                 │
  │                        │                             │                 │
  │  (poll)                │                             │                 │
  ├─ GET /api/v1/          │                             │                 │
  │   imports/[id] ────────┤                             │                 │
  │                        ├─ Return current status ─────┤                 ├─► import_jobs
  │◄───────────────────────┤ { status, counts, errors }  │                 │
```

**Endpoints involved**: `POST /api/v1/imports`, `GET /api/v1/imports`, `GET /api/v1/imports/[id]`
**Entities touched**: `import_jobs`, `user_stories`
**Feature IDs**: FEAT-IMP-001

**Why it matters**: BK-142 (staging credentials missing) blocks the entire import feature on staging. Async import with error-per-issue granularity means partial failures can go unnoticed.

---

## 4. Architecture behind the API

```
┌────────────────────────────────────────────────────────────────────┐
│                        Client Layer                                 │
│  Browser (Next.js SSR)    CLI (curl / bash)    AI agents (MCP)     │
└────────────────────────┬───────────────────────────────────────────┘
                         │
                         │ cookie (sb-*-auth-token) / bearer (bk_pat_*)
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer (Vercel / Next.js)           │
│                                                                     │
│  middleware.ts                                                      │
│  ├─ Route matcher (excludes /_next/*, /api/openapi)                 │
│  ├─ Supabase SSR refresh (cookie users)                             │
│  └─ Redirect /projects/* → /login if unauthenticated                │
│                                                                     │
│  lib/api/handler.ts (withApiHandler)                                │
│  ├─ Injects x-request-id, logging                                   │
│  ├─ resolveIdentity() → cookie or bearer auth                       │
│  ├─ Maps errors to canonical ErrorEnvelope                          │
│  └─ Enforces idempotency keys on mutations                          │
└────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                      Route Handlers (Next.js API routes)            │
│                                                                     │
│  /api/v1/me           → lib/api/principal.ts                        │
│  /api/v1/auth/*       → Supabase Auth SDK (server-side)             │
│  /api/v1/tokens       → lib/api/pat.ts                              │
│  /api/v1/workspaces   → workspace handlers + invites                │
│  /api/v1/projects     → project + module handlers                   │
│  /api/v1/modules      → module CRUD + move/delete cascade            │
│  /api/v1/user-stories → US + AC handlers                            │
│  /api/v1/atcs         → ATC handlers (version locked)               │
│  /api/v1/imports      → Jira import (async, jira.js)                │
└────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Supabase (PostgreSQL + Auth)                      │
│                                                                     │
│  RLS Policies: Every table has per-row security via:                │
│  ├─ bunkai_user_role(workspace_id) → viewer/member/admin/owner      │
│  ├─ bunkai_user_id() → resolves caller (cookie vs PAT)              │
│  ├─ bunkai_is_workspace_member() → boolean                          │
│  └─ bunkai_is_workspace_admin() → boolean                           │
│                                                                     │
│  Key tables (22 migrations):                                        │
│  workspaces, workspace_members, projects, modules,                  │
│  user_stories, acceptance_criteria, atcs, atc_steps,               │
│  atc_assertions, personal_access_tokens, workspace_invites,         │
│  import_jobs, activity_log, idempotency_keys, feature_flags,        │
│  magic_link_tokens, user_view_state                                 │
│                                                                     │
│  Critical RPCs (SECURITY DEFINER):                                  │
│  ├─ create_atc_v1 / update_atc_v1 — ATC + steps + assertions       │
│  ├─ save_atc_v1 — legacy upsert                                     │
│  ├─ move_module() — re-parent + path recompute                      │
│  ├─ soft_delete_module_cascade() — cascade archive                  │
│  ├─ set_user_story_ready_to_test() — serialized gate                │
│  ├─ reorder_acs() — position renumbering                            │
│  └─ bootstrap_first_workspace() — auto-create on signup             │
└────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────────┐
│                      External Integrations                          │
│                                                                     │
│  Resend → Transactional email (magic links, invites)                │
│  Jira   → Issue import via jira.js (async)                         │
│  Vercel → Deployment + serverless execution                         │
└────────────────────────────────────────────────────────────────────┘
```

| Component | Role | Persistence touched | Why it matters for QA |
|-----------|------|---------------------|-----------------------|
| `middleware.ts` | Auth gate + session refresh | Supabase Auth | Misconfigured matcher = unprotected routes; stale session = login redirect loop |
| `withApiHandler` | Error envelope + identity + idempotency | `idempotency_keys` | Missing idempotency key headers cause 428; double submits create duplicates |
| `resolveIdentity()` | Cookie vs PAT resolution | `personal_access_tokens` | PAT regression (BK-84, BK-92) breaks all agent/CLI workflows |
| RLS policies | Per-row authorization | All user tables | Policy gaps = privilege escalation (BK-117, BK-134) |
| SECURITY DEFINER RPCs | Escalated operations | Multiple tables | Bugs in RPCs bypass RLS entirely — blast radius is wide |

---

## 5. External integrations

| Service | Trigger | Direction | Failure mode (user-visible) | Journeys affected |
|---------|---------|-----------|-----------------------------|-------------------|
| **Supabase Auth** | `POST /auth/magic-link`, `POST /auth/signup` | Outbound sync | 502 → user sees "Failed to send email" / 429 → "Too many requests, try later" | Journey 1 (signup) |
| **Resend** | Supabase → Resend (email delivery) | Outbound async | Silent — Supabase OTP succeeds but email never arrives; user never receives magic link | Journey 1 (signup) |
| **Jira (jira.js)** | `POST /api/v1/imports` → async worker | Outbound sync | 142 → import fails instantly with `jira_unauthorized` on staging; partial import on token expiry mid-import | Journey 5 (import) |
| **Vercel** | All routes | Infra | 504 on cold start (serverless); function timeout on large imports; deployment rollback = API contract drift | All |

---

## 6. Cross-references

- **Data-map entities this API exposes**: `workspaces`, `projects`, `modules`, `user_stories`, `acceptance_criteria`, `atcs` (with `atc_steps`, `atc_assertions`), `personal_access_tokens`, `workspace_invites`, `import_jobs` → see `.context/business/business-data-map.md`
- **Feature-map features this API backs**: FEAT-AUTH-001/002, FEAT-WS-001/002, FEAT-PROJ-001, FEAT-MOD-001, FEAT-US-001, FEAT-AC-001, FEAT-ATC-001, FEAT-IMP-001, FEAT-ACC-001/002 → see `.context/business/business-feature-map.md`
- **OpenAPI spec**: `public/openapi.json` (build-time generated from Zod schemas via `lib/openapi/registry.ts`)
- **TypeScript types**: auto-generated via `bun run api:sync` → `api/schemas/*.types.ts`

---

## 7. Discovery gaps

- **PAT scope enforcement**: The OpenAPI spec and schema define scopes (`atc:read`, `atc:write`, `run:execute`, `workspace:admin`) but BK-117/BK-134 confirm `workspace:admin` is self-issuable by member-role users. Actual per-route scope checks not verified.
- **Route-level auth metadata**: The OpenAPI spec documents security schemes but individual paths lack `security:` declarations — auth requirements are enforced in handler code, not spec-declared.
- **Rate limits**: Schema mentions `rate_limited` error code but no documented limits per endpoint. Generic 100/600 req/min assumed from SRS.
- **Idempotency**: Some mutation paths use `idempotency_keys` but the exact set of idempotent endpoints is not documented.
- **SSE for import polling**: `GET /api/v1/imports/[id]` is described as "SSE-ready" but no SSE implementation verified — current implementation is pull-only.
- **Async import worker**: The import job runs as part of the request-response cycle (not a background worker) — long JQL queries may hit Vercel's 10s function timeout.
- **Module route parity**: No GET endpoint for individual project detail exists — project data is assumed returned inline from creation or listed via workspace context.
- **No ATC list endpoint**: `GET /api/v1/atcs` does not exist — ATCs are surfaced through the project workbench UI only.
