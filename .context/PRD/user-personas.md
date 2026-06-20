# User Personas — Bunkai (QA Context)

> Source: target `.context/PRD/user-personas.md`. Four personas — three human, one AI agent.
> Each persona includes QA-specific testing angles.

---

## Persona 1 — Elena Vargas, Senior QA Engineer

**Role**: Day-to-day power user. Writes ATCs, runs Tests, files Bugs.

### Goals (QA testing focus)
- Create reusable ATCs and chain them into Tests
- Execute manual runs, marking pass/fail/block per step
- File bugs from inside the runner with full context
- Edit an ATC and verify changes propagate to every Test using it

### Testing priority: CRITICAL
- Elena's flows cover the core MVP loop. Every feature she touches must work perfectly.
- Key test areas: ATC CRUD, ATC anchoring (US+AC enforcement), Test chaining, Run execution, Bug filing

### Pain points to test against
- ATC edit propagation is NOT visible in the UI until reloaded
- Step-by-step runner should persist state across browser refresh
- Bug filing should complete in < 90s (Journey 2 target)

---

## Persona 2 — Mateo Silva, QA Lead/Manager

**Role**: Configures projects, watches dashboards, reads reports.

### Goals (QA testing focus)
- Create/manage projects and module hierarchy
- View defect heatmaps and run history
- Filter runs by date, module, status
- Manage workspace members and roles

### Testing priority: HIGH
- Mateo configures the structure that Elena operates in
- Key test areas: Workspace/Project CRUD, Module hierarchy (nesting, soft-delete cascade), RBAC enforcement, Run history, Bug dashboard

### Pain points to test against
- Module tree should render < 800ms at 500 modules / 5000 ATCs
- Soft-delete should cascade correctly to descendant entities
- Role assignments should be enforced at API level (viewer cannot write)

---

## Persona 3 — Sara Iglesias, Full-Stack Developer

**Role**: Occasional user. Checks test coverage linked to her features.

### Goals (QA testing focus)
- Search ATCs by name/module
- View Test details and run history
- Link GitHub PRs to ATCs (future)
- Consume Bunkai output without learning the tool deeply

### Testing priority: MEDIUM
- Sara's flows are mostly read-only
- Key test areas: ATC search, Test detail view, Run history view

---

## Persona 4 — Karim, AI Test Agent (non-human)

**Role**: API consumer. Drives Runs end-to-end via REST.

### Goals (QA testing focus)
- Authenticate with Bearer token (PAT)
- Fetch Test contract (`GET /tests/ID?expand=atcs.steps`)
- Start Run (`POST /runs` with idempotency key)
- Report step results (`POST /runs/ID/steps/{step_id}/result`)
- File Bug (`POST /bugs`)
- Close Run (`POST /runs/ID/finish`)

### Testing priority: CRITICAL
- Karim represents the AI/future-proofing bet of the product
- Key test areas: API auth (PAT scopes), Idempotency keys, Envelope response shape (`{success, data, error}`), Error codes (401, 404, 409, 429)
- Every endpoint must be testable via curl/Postman without a browser session

### API expectations
- OpenAPI 3.x spec at `/api/openapi.json`
- Bearer token auth (workspace-scoped)
- Idempotency keys on POST endpoints
- Predictable response envelope
- Rate limits with `Retry-After`

---

## Cross-persona Notes

- Personas 1 + 4 share the same data model — a manual Run and an agent-driven Run produce indistinguishable shapes
- Persona 2's dashboards must aggregate across both execution modes transparently
- Persona 3 is the adoption multiplier — if developers find Bunkai useful, they spread it across teams
