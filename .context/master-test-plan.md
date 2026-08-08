```
  ╔══════════════════════════════════════════════════════════════════╗
  ║              BUNKAI — Master Test Plan                         ║
  ║        What to test in this system, and why it matters         ║
  ╚══════════════════════════════════════════════════════════════════╝
```

## Executive Risk Map

Bunkai's core value is **test case integrity** — ATCs must be created, updated, searched, and status-tracked without data loss or corruption. The most fragile areas are the **RPC-powered write paths** (ATC create/update, module move/archive, AC rebalancing) because they bundle multiple mutations in a single DB transaction with business-rule enforcement. Auth/authorization is the second-critical zone: dual auth modes (cookie + PAT) with RLS-only authorization means a single policy mistake exposes all data. The Jira import path is lower business risk because it's async and isolated, but its error-handling surface is large.

| Priority | Flow | Why it matters | Depends on / Affects |
|----------|------|----------------|----------------------|
| CRITICAL | ATC Create & Update | Data integrity loss corrupts the core entity. Silent failures in RPC transactions cause partial saves (steps but no assertions, ACs but no ATC). | Auth, Module tree, User Stories, ACs |
| CRITICAL | Auth & Authorization | Both auth channels collapse to the same RLS-gated Postgres client. A broken RLS policy exposes ALL workspaces. A broken session/PAT handler locks out ALL users. | EVERY flow |
| CRITICAL | Module Tree CRUD | Soft-delete cascades can orphan stories/ACs/ATCs. Rename rebuilds paths for entire subtrees — one bad UPDATE breaks navigation. Move with cycle-miss can corrupt the tree. | ATCs, User Stories, Project structure |
| HIGH | AC Management (create/move/delete) | Last-AC-revert auto-demotes stories. Negative-parking rebalance bugs cause position conflicts on concurrent writes. | User Stories, ATC anchoring, ready_to_test gate |
| HIGH | Workspace Membership & Invites | Role escalation (viewer → admin) or status bypass (suspended → active) leaks data. No-demotion rule on invite accept can be bypassed. | All workspace data |
| HIGH | Dual Auth Parity (Cookie + PAT) | PAT issuance is browser-only. PAT-scope enforcement (atc:write, etc.) in RPCs. A bug here lets CLI agents escalate privileges. | ATC CRUD, Import, Workspace admin |
| MEDIUM | Jira Import | Async, one-per-project. Edge cases: JQL parse failures, rate limits, partial imports, ADF conversion loss. | User Stories, ACs (append-only) |
| MEDIUM | Activity Log / Audit Trail | Immutable log — no delete. Bugs that skip activity_log insertion lose forensic traceability. | Compliance, debugging |

---

## What to Test First and Why

### 1. ATC Create & Update (CRITICAL)

**Why it matters:** The ATC is Bunkai's core entity. If a QA engineer creates an ATC and steps are lost, or an ATC is saved without AC bindings, the system has failed its primary purpose. Worse: silent partial saves (RPC inserts ATC row but fails on children) are data corruption you don't notice until test-execution time.

**What commonly breaks:**
- RPC transaction partially fails — ATC row created but steps/assertions/AC bindings missing
- Optimistic-lock version mismatch causes silent overwrite (PATCH without `X-If-Match` succeeds when it should reject)
- AC validation passes for ACs outside the User Story's scope (anchoring moat violation)
- Slug collision on concurrent create (unique constraint catches this — but the transaction then rolls back the whole thing, user sees 500)
- Module-path derivation fails when module was renamed concurrently (stale path in slug)

**Dependencies:** Auth (user must be workspace member with `atc:write`), Module tree (module must exist), User Stories (story must be in `ready_to_test` status — or should it? Check if draft stories allow ATC creation).

**What an experienced QA would check:**
- Create an ATC with 5 steps and 3 assertions, then GET it — verify all children returned in order
- Create 2 ATCs concurrently targeting the same module → both succeed with unique slugs
- PATCH with a stale version → 409 Conflict, not silent overwrite
- PATCH with valid version → children are fully replaced (old steps gone, new steps appear)
- Create without atc:write scope token → 403
- Create with ACs from a different User Story → 422 with clear error
- Delete (archive) an ATC → GET returns 404, admin query shows archived_at set

### 2. Auth & Authorization (CRITICAL)

**Why it matters:** Auth is the single gate to everything. Since Bunkai uses RLS as its ONLY authorization layer (no TS-level guard), a typo in an RLS policy leaks data across workspaces. And because cookie sessions and PAT tokens both collapse to the same `Principal` + Supabase client, a flaw in `resolveIdentity()` bypasses both.

**What commonly breaks:**
- RLS policy for `workspace_members` uses wrong workspace_id column → cross-tenant data leak
- `resolveIdentity()` fails for PAT tokens after DB secret rotation (token verification uses SHA-256 hash in sibling table)
- PAT tokens issued via cookie still work after explicit revocation (revoked_at check missed in RLS)
- Magic link tokens consumed twice (replay — the `consumed_at` check in the sibling table might not be atomic)
- PAT issuance via PAT (forbidden) — a CLI agent calling POST /tokens gets 403

**Dependencies:** Everything.

**What an experienced QA would check:**
- User A's PAT cannot read User B's workspace data across any endpoint
- Revoke PAT → immediate 401 on next request with that token
- Sign in, create PAT, sign out, use PAT → works (PAT is independent of session)
- Magic link consumed → second use returns error (not success)
- PAT with `atc:read` only → PATCH /atcs/{id} returns 403
- Workspace member = `viewer` → POST /modules returns 403

### 3. Module Tree CRUD (CRITICAL)

**Why it matters:** The module tree is the organizational backbone. Everything nests under it. Corrupt the tree and you orphan stories, ACs, and ATCs — or worse, you show QA engineers the wrong structure and they write tests in the wrong module.

**What commonly breaks:**
- Rename at root level rebuilds paths for thousands of descendants → timeout or partial update
- Soft-delete at depth 3 (with children at depth 4-6) → recursive CTE misses a branch, leaving orphaned modules
- Move creates a cycle (module A parent = B, B parent = A) → tree traversal loops
- Move exceeds max depth 6 → accepted when it should be rejected
- Concurrent rename + move → race condition leaves stale path materialization

**Dependencies:** Workspace → Project must exist, user must be member.

**What an experienced QA would check:**
- Create module at depth 6 → success
- Create module at depth 7 → rejection
- Rename a mid-level module → ALL descendant paths update (verify by GET)
- Delete (archive) a module with children → children + stories + ATCs all archived (GET individual)
- Move root-level module under a leaf → cycle rejected
- Create two modules with same slug (different parent) → allowed (path is unique, not slug)

### 4. AC Management (HIGH)

**Why it matters:** ACs are the bridge between "what the business wants" (User Story) and "what we test" (ATC). Losing an AC or corrupting its position breaks that bridge — and the auto-revert of story status on last-AC-delete is a silent semantic change.

**What commonly breaks:**
- Concurrent AC insert at same position → both race to position N, one wins, one gets duplicate (negative-parking should prevent this — verify under load)
- Delete second-to-last AC → story stays ready_to_test (correct). Delete last AC → story reverts to draft (automatic)
- Move AC from position 1 to position 5 → positions 2-5 shift up correctly
- AC soft-delete (archive) → not visible in standard queries but recoverable

**Dependencies:** User Story, Module.

**What an experienced QA would check:**
- Create 5 ACs on a story, verify positions 1-5 consecutive
- Insert at position 3 → new AC is pos 3, old 3 → 4, 4 → 5, 5 → 6
- Delete all ACs → story.status = "draft" (verify via GET story)
- Try to set story to ready_to_test with 0 ACs → rejection
- Concurrent insert at same position → both succeed, no collision

---

## State Machines That Matter

### ATC Status

**Why it matters:** ATC status drives test execution reporting. An ATC stuck in `running` (runner crash) inflates "in progress" counts. A transition from `blocked` → `pass` without re-running is suspicious.

**Transitions most likely broken:**
- `running → blocked` (does the runner handle dependency failures gracefully?)
- `blocked → unrun` (reset not offered in UI, but valid at DB level)
- Direct jump `unrun → pass` (bypasses execution — possibly valid for pre-asserted ATCs?)

**Forbidden states to guard:** None enforced at DB level (any → any is allowed). If the product wants lifecycle restrictions (e.g., `blocked` cannot go to `pass` without re-running), enforce at app layer and test the gate.

**Detection:** A sudden spike of ATCs in `running` across a project suggests the execution runner has a bug.

### User Story Status

**Why it matters:** `ready_to_test` is the signal that a story is groomed enough for ATC authoring. If a story stays `ready_to_test` with 0 ACs (the auto-revert fails), engineers waste time clicking into an empty story.

**Transitions most likely broken:**
- `ready_to_test → draft` on last AC delete (the DB function fires but the returned response shows stale status)
- `ready_to_test` set with exactly 0 ACs (the FOR UPDATE lock should prevent this — but two concurrent transactions: one deletes AC, other sets ready_to_test)

---

## Silent Killers — Automated Processes

### Vercel `after()` Jira Import

**What it does:** After the HTTP 202 response, Vercel's `after()` slot runs the JQL import in background. It pages through Jira API, creates/updates stories and ACs in Supabase.

**What breaks if it fails:**
- Misses a page → partial import (some stories missing). No user notification.
- Runs twice → duplicate stories/ACs (idempotency depends on Jira issue ID matching — if JQL changes, duplicates happen)
- Runs out of Vercel timeout → import marked `failed` but partial data is already committed.

**How failure is detected today:** Status poll via GET `/api/v1/imports/{id}`. If the user doesn't poll, they never know it failed. No email/webhook on failure.

**Recommended QA strategy:**
- Synthetic probe: create an import with a known JQL that returns 2-3 issues, poll until complete, verify exact count
- Edge case: JQL returning 0 results → import succeeds with 0 rows
- Edge case: JQL syntax error → import fails fast with error in `errors` field
- Concurrency: try to start a second import while one is running → should reject (partial unique index)

### DB Triggers (atcs_set_updated_at, atcs_refresh_tsv)

**What they do:** Auto-set `updated_at` on any ATC update. Auto-refresh the `tsvector` for full-text search on title/tag changes.

**What breaks:** Disabled trigger (DB migration, manual SQL) — silently. `updated_at` stops updating; TSV search returns stale results. No one notices until someone searches for a newly-created ATC title and it doesn't appear.

**Detection:** None — no monitoring on trigger health.

**Recommended QA strategy:**
- After every ATC create/update, verify `updated_at` is within 1 second of the response timestamp
- After every ATC title update, search for the new title term and confirm it appears in results

### AC Auto-Revert on Last-AC-Delete

**What it does:** When the last active AC is archived, the DB function sets `user_stories.status = 'draft'`.

**What breaks:** The function errors (e.g., FK constraint on ATCs still referencing that AC) but the AC archive already committed → story stays `ready_to_test` with 0 ACs. Or the function silently fails and the story appears ready when it's not.

**Detection:** None — the UI might show a story as ready_to_test with no ACs displayed.

---

## External Integrations — Failure Points

### Supabase

| Failure mode | Business impact | Degradation |
|-------------|----------------|------------|
| DB down | Complete app outage. No auth, no CRUD, no search. | None — Bunkai is a DB-first app |
| RLS policy misconfiguration | Cross-tenant data leak OR full lockout | None |
| Auth service degraded | New logins fail; existing sessions may work until cookie expiry | Sessions cache briefly (cookie TTL), but no login |
| Connection pool exhausted | Random 5xx on high traffic | None documented |

**Critical timeouts:** Default Supabase client timeout. On slow network, requests may hang. Recommend verifying explicit timeout config and testing with a simulated slow DB.

### Jira

| Failure mode | Business impact | Degradation |
|-------------|----------------|------------|
| Rate limited | Import pauses/resumes | Non-blocking — import retries on next run? (unclear from code — check retry logic) |
| API key expired | All imports fail with auth error | Import job fails with clear error; existing data unaffected |
| API schema change | ADF→Markdown conversion produces garbled AC content | Content loss, but structure intact |

**Known quirks:**
- JQL syntax differs between Jira Cloud and Jira Server (Bunkai targets Cloud)
- Pagination limit of 1000 pages (1000 results? 1000 requests? — verify the actual cap)
- ADF markdown conversion may lose formatting (bullets, tables, code blocks)

---

## Dependency Cascade Between Flows

```
Auth ──► Workspace ──► Project ──► Module Tree ──► User Stories ──► ACs ──► ATCs
  │          │             │              │               │           │        │
  └── all flows die if auth fails ───────┘               │           │        │
                                                          │           │        │
                                           Last AC delete ──► reverts story  │
                                                                             │
                                           ATC anchoring ──► must reference ◄┘
                                                              ≥1 active AC
```

**Critical chain 1 — ATC create depends on everything upstream:**
```
Auth → Workspace membership → Project access → Module exists → Story exists → ACs exist
```
Break any link and the user cannot create an ATC. Test the error messages for each missing link — the user needs to know WHICH dependency is missing, not a generic 403/500.

**Critical chain 2 — AC delete cascades to story status:**
```
Delete AC → no remaining active ACs → story reverts to draft → story no longer ready_to_test
```
This is correct behavior but can surprise users. Verify the UI reflects the status change within the same page load (no stale cache).

---

## Edge Cases Developers Commonly Forget

### Concurrency

| Theme | Flow at risk | What to test |
|-------|-------------|--------------|
| AC position race | AC create/move | Two users insert at same position → both succeed, no collision (negative-parking) |
| ATC create race | ATC create | Two users create ATC with same module-path → both succeed with unique slugs |
| Module rename vs create | Module tree | Rename module while another user creates a child → path consistency |
| Story status race | User Stories | Concurrent: user A deletes last AC, user B sets ready_to_test. One should fail. |

### Data Limits

| Theme | Flow at risk | What to test |
|-------|-------------|--------------|
| Module depth | Module tree | 6 levels max → 7th level rejected |
| ATC steps/count | ATC create | 100 steps on one ATC → verify response time and ordering |
| Module children | Module tree | 500 sibling modules under one parent → rename performance |
| Slug length | ATC create | Very long module path + long slug → truncation or 413? |
| Jira import size | Import | JQL returning 10,000 issues → import handles pagination |

### Permission Boundaries

| Theme | Flow at risk | What to test |
|-------|-------------|--------------|
| Cross-workspace | All | User A from workspace 1 cannot access workspace 2's data even with direct URL |
| Viewer write | Module, AC, ATC | Viewer role → all write endpoints return 403 |
| Token scope | ATC | atc:read token → PATCH returns 403 |
| PAT issuance | Auth | PAT trying to create PAT → 403 |
| Revoked token | Auth | Revoked PAT → 401 on next request |
| Invite demotion | Workspace | member receives viewer invite → reject (no demotion rule) |

### Orphaned Data

| Theme | Flow at risk | What to test |
|-------|-------------|--------------|
| Module archive | Module tree | Archive module → stories not accessible via module tree, but direct URL to story should 404 |
| ATC archive | ATC | Archive ATC → removed from list/serach, but activity log entry exists |
| User removal | Workspace | Remove user from workspace → their PATs still work? (should they be revoked?) |

---

## Pre-Release Checklist (Priority-Ordered)

1. `[CRITICAL]` Verify ATC create with 5 steps + 3 assertions + 2 AC bindings returns all children in correct order
2. `[CRITICAL]` Verify PATCH ATC with stale `X-If-Match` header returns 409
3. `[CRITICAL]` Verify User A's PAT returns 403 for User B's workspace data across all endpoints
4. `[CRITICAL]` Verify viewer-role member receives 403 on POST/PATCH/DELETE endpoints
5. `[CRITICAL]` Verify revoked PAT returns 401 on next request
6. `[CRITICAL]` Verify module create at depth 7 returns 422/400
7. `[CRITICAL]` Verify module delete cascades to archive children + stories + ACs + ATCs
8. `[HIGH]` Verify concurrent AC insert at same position — both succeed, no collision
9. `[HIGH]` Verify last-AC-delete reverts story status to `draft`
10. `[HIGH]` Verify story cannot transition to `ready_to_test` with 0 ACs
11. `[HIGH]` Verify PAT with `atc:read` only cannot PATCH/DELETE ATCs
12. `[HIGH]` Verify invite accept with lesser role than existing membership is rejected
13. `[MEDIUM]` Verify Jira import with 0 results reports success with 0 rows
14. `[MEDIUM]` Verify concurrent Jira import on same project rejects the second
15. `[MEDIUM]` Verify full-text search finds ATCs created in the current session

---

## What is NOT in This Plan

- Flow-level diagrams and state-machine transition tables → `.context/business/business-data-map.md`
- Feature catalog, CRUD matrix, feature flags → `.context/business/business-feature-map.md`
- API endpoint inventory / contracts → `bun run api:sync` + `/business-api-map`
- Detailed test case definitions and traceability → TMS (see `/test-documentation`)
- Sprint-level execution order → `.context/reports/SPRINT-{N}-TESTING.md` (see `/sprint-testing`)
- UI-specific test scenarios (layout, responsive, dark mode, accessibility) → per-ticket testing via `/sprint-testing`
- Performance/load testing — not in scope for MVP
- Database migration testing — assumes migrations are tested separately

---

## Discovery Gaps

- **Feature map generated** — `.context/business/business-feature-map.md` exists and was cross-referenced. Enabled CRUD-coverage gap analysis (module tree read endpoints undocumented, workspace delete/member-role-change missing), per-feature QA-relevance tagging in §Executive Risk Map, and the zero-coverage baseline in §Pre-Release Checklist.
- **Live API not exercised** — endpoint behavior derived from route files and DB functions. Actual error responses, status codes, and edge-case behavior may differ.
- **RLS policies not inspected** — authorization model derived from code, not from `pg_policies` inspection. The actual Postgres-level gates may differ from what the code expects.
- **Retry logic on Jira import** — unclear if failed imports auto-retry or require manual re-trigger.
- **Vercel `after()` timeout** — unclear what happens if the import exceeds the background slot timeout. Does it fail gracefully?
- **ATC status lifecycle enforcement** — DB allows any→any transitions. If the product intends restrictions (e.g., `blocked → pass` requires re-run), the gate is not yet implemented.
