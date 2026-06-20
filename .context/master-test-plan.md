# Master Test Plan — Bunkai (upex-bunkai-tms)

> What to test in this system, and why | Generated: 2026-06-20

```
╔══════════════════════════════════════════════════════════════╗
║                    BUNKAI — Master Test Plan                ║
║  "Decomposes user stories into executable ATCs"             ║
║  Pre-MVP · Testing Maturity: 0/4 · Zero automated tests    ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 1. Executive risk map

Bunkai is a pre-MVP greenfield project with 0% test coverage, 26 open bugs, and incomplete PAT scope enforcement. The biggest risk is that its core invite-and-workspace integrity does not hold (BK-60/61/62 — duplicate invites, role overwrite), followed by a PAT security leak (BK-117/134 — `workspace:admin` self-issuable by members). Everything else is isolated CRUD — important, but the blast radius is narrower.

| Priority | Flow | Why it matters | Depends on / Affects |
|----------|------|----------------|----------------------|
| CRITICAL | Workspace invites & roles | Data integrity — duplicate invites and role overwrite (BK-60/61/62) can give any invitee admin access or lock out legit users | Auth, workspace membership, all gated features |
| CRITICAL | PAT scope enforcement | Security — BK-117/134 lets member-role users self-issue `workspace:admin` tokens, bypassing RLS | All API routes (bearer auth) |
| CRITICAL | Auth & session management | Everything is gated behind auth — if magic-link or PAT auth breaks, the system is unreachable | Every feature |
| HIGH | Module hierarchy operations | BK-57 (rename+move not atomic), BK-67 (depth >=5 toast suppressed), BK-68 (1-char names) — data corruption risk | Projects, user stories, ATCs |
| HIGH | ATC authoring (create + edit) | Core value proposition — BK-96 (PATCH returns 412), optimistic locking version drift | Tests, runs, traceability |
| HIGH | Jira import | BK-142 (staging credentials missing) — async import can partially fail without alert | User stories, modules |
| MEDIUM | User story ready-to-test gate | Serialized lock (`set_user_story_ready_to_test`) — concurrent calls could bypass AC requirement | ATCs, traceability |
| MEDIUM | Markdown editor validation | BK-99 (50KB limit not enforced), BK-100 (90% warning not implemented) — server-side bypass | User story descriptions |
| LOW | API docs (Scalar) | Cosmetic — stale OpenAPI spec only matters if consumers rely on it | Developer/QA tooling |

---

## 2. What to test first and why

### CRITICAL — Workspace invites & roles (FEAT-WS-002)

**Why it matters**: BK-60 (no email uniqueness against active members), BK-61 (no check against pending invites), and BK-62 (role overwrite via upsert) mean invite integrity is broken. A member-role user who accepts an admin's invite can be demoted by the upsert. A bad actor can send duplicate invites to exhaust tokens.

**What commonly breaks**:
- `POST /workspaces/[id]/invites` with same email as an active member → creates duplicate invite (should reject)
- `POST /workspaces/[id]/invites` with same email as a pending invite → creates second invite (should return existing)
- Accepting an invite as viewer when workspace already has a member-role record → upsert overwrites role to whatever the invite says (BK-62)
- Rotating an invite that was already accepted → should error, currently might succeed
- Invite token expiry not enforced on accept

**Dependencies**: Auth (caller must be workspace admin), workspace existence

**What an experienced QA would check**:
- Invite the same email twice — second should 409. Then invite someone who is already a member — should 409.
- As admin, invite as "viewer". As the invitee, accept. Then check `workspace_members.role` is "viewer" (not overwritten). Then have a different admin invite that same user as "admin" and re-accept — verify role is still "viewer" (should require explicit leave-first).
- Try to invite as a non-admin (member) — should 403. Try to invite from outside the workspace — 403.
- Accept an expired invite token — should fail with 410 Gone.

### CRITICAL — PAT scope enforcement (FEAT-AUTH-002)

**Why it matters**: BK-117 and BK-134 confirm that any authenticated user (including `member` role) can issue a PAT with `workspace:admin` scope. This defeats RLS — a bearer token with admin scope can call admin-gated endpoints without being an admin member.

**What commonly breaks**:
- `POST /api/v1/tokens` as member-role user with `scopes: ["workspace:admin"]` → should 403, currently issues the token
- Using that `workspace:admin` PAT to invite new members, delete modules, access other workspaces
- Token revocation not actually enforced at the middleware level
- Scope filtering in `resolveIdentity()` — the `auth.scopes` field in `GET /me` response might lie

**Dependencies**: Auth middleware, PAT minting logic

**What an experienced QA would check**:
- As a member-role user (not admin), POST a token with `workspace:admin` — expect 403, confirm it 200s (bug confirmed)
- Then use that token to call admin-gated endpoints — expect 403, confirm what actually happens
- Revoke the token, then use it again — should 401
- Create a token with no scopes — should 422. With `atc:read` only — confirm it cannot POST ATCs (403)

### CRITICAL — Auth & session management (FEAT-AUTH-001)

**Why it matters**: If users cannot sign up, sign in, or maintain a session, the entire application is dead. Two auth paths exist (cookie + PAT) and regression BK-84 confirms PAT auth was broken on staging at least once.

**What commonly breaks**:
- Magic-link email never arrives (Resend delivery failure, Supabase rate limit hit: 4/hour/email on free tier)
- Session cookie not refreshed by middleware → redirect loop on `/projects`
- PAT bearer rejected on endpoints that worked before (BK-84 — middleware regression)
- Signup creates user but fails to create first workspace (bootstrap trigger race)
- Callback URL mismatch between dev/staging/production environments

**Dependencies**: Supabase Auth, Resend, middleware.ts

**What an experienced QA would check**:
- Full signup flow: POST magic-link → check email → click link → verify cookie set → access `/projects`
- Refresh the session after cookie expiry → middleware should refresh transparently
- Signup via `POST /auth/signup` → verify PAT returned + workspace auto-created
- Use PAT on every API endpoint — verify no route accidentally rejects bearer (BK-84 regression pattern)
- POST to `/api/v1/me` with no auth → should 401

### HIGH — Module hierarchy operations (FEAT-MOD-001)

**Why it matters**: BK-57 confirms that `PATCH /modules/{id}` with both `name` and `parent_module_id` is not atomic — the two RPC calls (rename + move) can partially fail. BK-67 suppresses the depth-warning toast at depth >=5. BK-68 allows 1-char names client-side.

**What commonly breaks**:
- Rename+move in same PATCH → module renamed but not moved (or vice versa), leaving inconsistent state
- Move a module under its own descendant → cycle should be rejected, might succeed and break path recompute
- Create module at depth 6 (max) → should get warning. Try depth 7 → should 400
- Delete a module with 50+ descendants → cascade timeout on serverless function (Vercel 10s limit)

**Dependencies**: Projects, materialized path logic, `move_module()` RPC, `soft_delete_module_cascade()` RPC

**What an experienced QA would check**:
- PATCH rename+move together 20 times — verify at least one case where name changes but parent_module_id stays (race condition). Check both fields in the response.
- Move module A under module B, then move module B under module A → cycle should 400
- Create 6 levels of nesting, check warning appears. Try level 7 → should 400
- Soft-delete a module with 3 user stories and 5 ATCs → verify all descendants archived. Restore is NOT implemented — confirm no restore endpoint exists.
- Create module with 1-char name → should return 422 (client-side check is not server-side safety)

### HIGH — ATC authoring (FEAT-ATC-001)

**Why it matters**: ATCs are the product's core entity. BK-96 (PATCH returns 412 instead of 200 on happy path) means the edit confirmation is broken — the edit commits but the response says precondition failed. Optimistic locking (`version` column) means concurrent edits silently lose.

**What commonly breaks**:
- Happy-path PATCH returns 412 (BK-96) — user thinks their edit failed but it actually committed
- Stale version update returns 409 — correct, but error message may not explain "refresh and retry"
- Full-replace semantics: omitting a field in the body might clear it (API expects all fields)
- Creating ATC with 0 steps → should 422. With 0 acceptance_criterion_ids → should 422
- `create_atc_v1()` RPC is SECURITY DEFINER — mistakes in the RPC bypass RLS

**Dependencies**: Modules, user stories, ACs

**What an experienced QA would check**:
- Create ATC, read it back, verify slug format `<module-slug>/atc-<8hex>`
- Edit ATC → confirm 200 response (not 412). Read back and verify edit committed
- Edit with stale `version` → confirm 409
- Omit `steps` array → 422. Omit `acceptance_criterion_ids` → 422
- Create ATC with layer=UI, then change to layer=API on edit — verify persisted
- Verify `atc_acceptance_criteria` junction table has correct rows after create

### HIGH — Jira import (FEAT-IMP-001)

**Why it matters**: BK-142 (staging credentials missing) means the entire import feature is dead on staging. The async model with per-issue error granularity means partial failures can easily go unnoticed.

**What commonly breaks**:
- Import fails instantly with `jira_unauthorized` (BK-142) — no graceful degradation
- Import processes 47/50 issues then hits rate limit — remaining 3 silently disappear from result
- Importing the same JQL twice → `imported_count` should show "already imported, 0 created" but may double-create if external_id dedup fails
- One-active-import constraint should reject a second import on the same project while first is running

**Dependencies**: jira.js, Jira REST API, Supabase DB

**What an experienced QA would check**:
- POST import with invalid JQL → get job back with status "queued", then poll until it fails with error details
- POST import with valid JQL → poll until "completed", verify `created_count` matches expected
- Wait for import to complete, then import again — `created_count` should be 0, `updated_count` may show matching
- Start an import, immediately start another on same project → should 409 (one-active-import constraint)
- Verify imported user stories have `external_id` set and `external_url` pointing to Jira

---

## 3. State machines that matter

### Module soft-delete cascade

```
Module → archived_at = NOW()
  → child Modules → archived_at = NOW()
  → User Stories (module-level) → archived_at = NOW()
  → ATCs → archived_at = NOW()
```

**Why it matters**: The cascade runs inside a SECURITY DEFINER RPC (`soft_delete_module_cascade()`). If it errors partway through, some descendants remain active while the parent is archived — orphaned data with no UI to surface it.

**Transitions most likely to be broken**:
- Cascade with 100+ descendants → Vercel function timeout (10s) mid-cascade
- Cascade on a module that was already soft-deleted → idempotent or error?
- Cascade on a module whose parent was already soft-deleted → should cascade further up? (It should not — deletion goes downward)

### Run state machine (NOT YET IMPLEMENTED)

```
[*] → created → running → passed → [*]
            ↓
      ┌→ failed → [*]
      └→ aborted → [*]
```

**Why it matters**: The Run state machine is fully planned (BK-34–BK-39) but not implemented. When it lands, the most critical invariant is: a Run cannot be finished twice, and step results cannot be posted after the Run is finished.

### User story ready-to-test gate

```
draft ──────────────────────► ready_to_test
  (set_user_story_ready_to_test() validates:
   - At least 1 active AC exists
   - Gate is serialized via FOR UPDATE)

ready_to_test ─────────────► draft
  (re-opening allowed)
```

**Why it matters**: SERIALIZABLE FOR UPDATE gate prevents concurrent bypass of the AC requirement. A race condition here means an empty user story can be flagged ready-to-test.

**Transitions most likely to be broken**:
- Two concurrent PATCH requests to set `ready_to_test` — one should block and get the updated state
- Set `ready_to_test` with 0 active ACs → 409. Add an AC, retry → 200. Archive the AC, try to set ready → 409
- Direct DB insert bypassing the RPC — RLS on `user_stories` should reject setting `status = 'ready_to_test'` without calling the RPC

---

## 4. Silent killers — automated processes

| Process | What it does | Breaks if | Detection today | Recommended QA strategy |
|---------|-------------|-----------|-----------------|------------------------|
| `bootstrap_first_workspace()` trigger | Auto-creates workspace on auth.users INSERT | Trigger disabled, RLS misconfigured, or race condition on signup | None — user sees empty workspace list with no error | Synthetic signup → verify workspace exists in GET /me response |
| `import_jobs` async processing | Fetches Jira issues, upserts user stories | Vercel timeout (10s), Jira API rate limit, token expiry mid-import | None — job status shows "running" forever (no timeout) | Poll import job every 5s until terminal; log timeout as failure |
| `idempotency_keys` TTL cleanup | Removes old idempotency keys from DB | No cleanup → table grows unbounded | None — no monitoring on idempotency_keys table size | Verify mutation with same key returns cached response; check TTL is honored |

---

## 5. External integrations — failure points

| Service | Business flow that stops | Acceptable degradation | Known quirks |
|---------|-------------------------|----------------------|--------------|
| **Supabase Auth** | Signup, sign-in, session refresh, PAT identity resolution | None — hard fail on every auth-dependent flow | Free tier: 4 OTP emails/hour/email (429). Admin API requires `service_role` key |
| **Resend** | Magic-link delivery, invite emails | None — user never receives OTP or invite. No fallback provider | Delivery time varies (2s–2min). No tracking on recipient open/click |
| **Jira (jira.js)** | Jira import | Partial degradation — import runs but may miss issues. BK-142: staging completely dead | Rate limits per API token. Pagination required for >50 issues per JQL |
| **Vercel** | All API routes + frontend | Cold start latency (>1s on infrequent routes). Function timeout (10s default) for heavy imports | Canary Next.js 15 — build-only bugs possible. OpenAPI spec is force-static (no runtime update) |

---

## 6. Dependency cascade between flows

```
Signup ──► Auth ──► Workspace ──► Invite ──► Member joins
  │          │          │            │
  │          │          ▼            ▼
  │          │       Project      Workspace
  │          │          │         Membership
  │          │          ▼
  │          │       Module tree
  │          │          │
  │          │          ▼
  │          │     User Story ──► AC ──► ATC ──► Test (chain) ──► Run ──► Bug
  │          │                       │        │         │           │       │
  │          │                       │        │         │           │       │
  │          │                       ▼        ▼         ▼           ▼       ▼
  │          │                    Traceability    Coverage         Defect Heatmap
  ▼          ▼
PAT ──── Scope enforcement ────► All API routes
```

The critical insight: **Auth and workspace membership are the root dependencies of everything else**. If BK-60/61/62 compromise invite integrity, the cascade means every downstream feature (projects, modules, ATCs, runs, bugs) operates on corrupt membership data. Similarly, if PAT scope enforcement is broken (BK-117/134), a leaked member-level PAT can escalate to admin on any workspace.

---

## 7. Edge cases developers commonly forget

### Concurrency
- Two concurrent `PATCH /modules/{id}` calls — one renames, one moves. BK-57: they are not atomic within the same call, let alone across concurrent calls
- Two concurrent `set_user_story_ready_to_test()` — FOR UPDATE should serialize, but application-level retry logic is unverified
- POST import on same project while another is running — the one-active-import constraint is a DB-level check, what happens when the app sends two in rapid succession?

### Data limits
- Module name: min 2 chars per API spec, but BK-68 confirms 1-char names pass client-side
- User story description: API spec says max 50KB, BK-99 confirms server-side does not enforce it
- ATC steps: max 2KB per step content, max 10 tags per ATC
- Module nesting: max depth 6, but `path` column is text — what happens at depth 20?

### Permission boundaries
- `workspace:admin` scope on PAT issued by a member-role user — BK-117, BK-134
- Cross-workspace access: can a PAT scoped to workspace A read data from workspace B?
- Archived entities: can a viewer-role user see archived data? (They should not, per soft-delete design)
- RLS bypass via SECURITY DEFINER RPCs — `create_atc_v1()` and `soft_delete_module_cascade()` run with elevated privileges

### Idempotency
- Which mutation endpoints actually enforce idempotency keys? OpenAPI spec mentions `idempotency_key_required` error but per-endpoint documentation is missing
- Double-click on "Create Project" without idempotency key → 2 projects with same name/slug? (Slug uniqueness per workspace should prevent this)
- Retry with same key after timeout → verify cached response is returned, not a new execution

---

## 8. Pre-release checklist (priority-ordered)

1. **Verify invite integrity**: invite same email twice → 409. Invite existing member → 409. Accept invite → verify role is not overwritten (BK-60/61/62)
2. **Verify PAT scope enforcement**: member-role user tries `POST /tokens` with `workspace:admin` → 403 (BK-117/134)
3. **Verify signup flow**: magic-link → callback → session cookie → access protected page
4. **Verify PAT auth**: signup → use PAT on every API endpoint → no 401 regression (BK-84 pattern)
5. **Verify module rename+move atomicity**: PATCH with both `name` and `parent_module_id` → both applied or both rejected (BK-57)
6. **Verify ATC edit response**: PATCH an ATC → confirm 200, not 412 (BK-96)
7. **Verify Jira import on staging**: credentials configured → import completes without `jira_unauthorized` (BK-142)
8. **Verify module depth enforcement**: create at depth 6 → warning. Try depth 7 → 400 (BK-67)
9. **Verify user story ready-to-test gate**: set ready with 0 ACs → 409. Add AC, retry → 200
10. **Verify soft-delete cascade**: delete module → confirm all descendants archived. Verify they disappear from UI
11. **Verify Markdown size limits**: upload >50KB → 422 (BK-99). Check warning at ~45KB (BK-100)
12. **Verify cross-workspace isolation**: PAT from workspace A cannot read data from workspace B
13. **Verify idempotency**: POST create with same key → same response, no duplicate entity
14. **Verify rate limit**: exceed 600 req/min → 429 with `rate_limited` code
15. **Verify activity_log audit writes**: module rename/move/delete → verify row in `activity_log` (BK-59)

---

## 9. What is NOT in this plan

- Flow-level diagrams and state-machine transition tables → `.context/business/business-data-map.md`
- Feature catalog, CRUD matrix, feature flags → `.context/business/business-feature-map.md`
- API endpoint inventory / contracts → `bun run api:sync` + `.context/business/business-api-map.md`
- Detailed test case definitions and traceability → TMS (see `/test-documentation`)
- Sprint-level execution order → sprint testing workflow (see `/sprint-testing`)

---

## 10. Discovery gaps

- The feature-map was generated from code scan and PBIs, not from live testing — some CRUD gaps may be incomplete.
- Jira import's actual error handling at scale (500+ issues) is untested — Vercel timeout behavior on large imports is unknown.
- PAT scope enforcement scope (BK-117/134) was confirmed by code scan but actual blast radius (which routes are gated by which scopes) needs live testing.
- The `activity_log` table exists and BK-59 (module audit writes) is a known gap — it is unclear which other operations write to activity_log.
- No performance testing has been done — cold start latency, DB query plans, and RPC execution time are unknown.
- The Run state machine and related tables are not implemented yet — when they land, the test plan needs a new CRITICAL section for execution integrity.
