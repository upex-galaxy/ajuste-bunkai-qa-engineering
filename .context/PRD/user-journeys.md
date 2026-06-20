# User Journeys — Bunkai (QA Context)

> Source: target `.context/PRD/user-journeys.md`. Three journeys. Each includes testing angles.

---

## Journey 1 — "First-time setup: from sign-up to first ATC"

### Persona: Elena Vargas (Senior QA Engineer)

### Steps (what to test)

| Step | Action | Expected | Test Notes |
|---|---|---|---|
| 1 | Sign in with GitHub OAuth or magic-link email | 200, user created, redirect to Workspace Home | Test OAuth callback + magic-link fallback |
| 2 | Workspace Home loads | 4 stat cards (0s), "Create your first Project" CTA | Empty-state rendering |
| 3 | Create Project ("Checkout v2") | 201, land in Project View with empty tree | Edge: duplicate name, special chars |
| 4 | Create Modules ("Cart", "Payment", "Confirmation") | Tree renders with IDs (MOD-001...), grey status dots | Test nesting depth limit (6), soft warning at 4 |
| 5 | Import User Story from Jira | ACs parsed from Jira description | Test with malformed ACs, missing Jira credentials |
| 6 | Create first ATC anchored to AC-01 | Preview renders Markdown, save persists | Test layer=UI/API/Unit enforcement, missing layer blocked |
| 7 | Command palette (Cmd+K) → "create ATC" | New ATC form scoped to same Module | Keyboard-first workflow |

### Alternative paths

- GitHub OAuth blocked by IT → magic-link fallback within 30s
- Workspace invite expired → "Your invite has expired — ask {inviter_email} for a new one"
- Jira credentials missing → "Import from Jira" hidden, replaced by "Add Workspace Integration"
- No Modules created → ATC creation blocked; form selects from existing Modules only

---

## Journey 2 — "Manual execution: running a regression test + filing a bug"

### Persona: Elena Vargas, day 3 with Bunkai

### Steps (what to test)

| Step | Action | Expected | Test Notes |
|---|---|---|---|
| 1 | Find Test, filter "smoke", click TEST-008 | Table View filter, Test detail with 7 ATCs | Persisted view state across sessions |
| 2 | Start Run, pick environment "staging" | Run created `running`, redirected to Run screen | Empty environment dropdown handled |
| 3 | Walk through ATCs, mark pass | Status dot green, progress "1/7" | Keyboard shortcuts P/F/B |
| 4 | Hit failure on ATC-005, mark step fail | "Report Bug" CTA highlights | Pre-filled Bug form with Module/ATC context |
| 5 | File Bug (severity P2) | BUG-014 created, linked to Run + ATC + Module | Test Jira sync if configured |
| 6 | Abort Run with reason | Status `aborted`, coverage dashboard updates in 5s | Duplicate abort idempotent |
| 7 | Mateo sees abort on Dashboard | Heatmap shows Payment Module defect count +1 | Real-time propagation |

### Edge cases

- Browser refresh during Run → state persisted, resume from same step
- Two QAs run same Test simultaneously → two separate Run records
- Bug filing fails (Jira sync down) → saved natively with `pending_jira_sync`
- Network drop mid-Run → last-saved step durable

---

## Journey 3 — "AI agent runs a Test via API"

### Persona: Karim (AI Test Agent)

### Steps (what to test)

| Step | Agent Action | Expected | Test Notes |
|---|---|---|---|
| 1 | `GET /api/v1/me` with Bearer token | 200, scopes confirmed | 401 with `code: TOKEN_EXPIRED` on expiry |
| 2 | `GET /api/v1/tests/TEST-008?expand=atcs.steps` | Full payload with steps + assertions | Edge: test ID not found (404) |
| 3 | `POST /api/v1/runs` with idempotency_key | 201, `run_id: RUN-451`. Retry = same ID | Idempotency collision test |
| 4 | `POST /runs/RUN-451/steps/{step_id}/result` | 200, run state updated | Realtime push to UI (Supabase) |
| 5 | `POST /api/v1/bugs` on failure | 201, BUG-015 linked to Run + ATC + Module | Rate limit (429) handling |
| 6 | `POST /api/v1/runs/RUN-451/finish` | 200, Run finalized, metrics computed | Double-finish idempotent? |

### API contract rules (verify each)
- Every POST endpoint accepts idempotency key
- Response envelope: `{ success, data, error }`
- Error responses include machine-readable `code` field
- Rate limits: 100 req/min write, 600 req/min read
