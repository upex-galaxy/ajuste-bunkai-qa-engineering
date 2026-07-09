# Shift-Left Refinement: BK-50 — TMS-Traceability | Export the assembled chain as a read-only snapshot

**Status**: Refined — Awaiting PO Estimation
**Mode**: Shift-Left (pre-sprint, batch grooming)
**Refined on**: 2026-07-09
**Refined by**: QA — Shift-Left batch session
**Modality**: Jira-native

---

## Phase 1 — Critical Analysis

### Business context
- **Primary persona affected**: QA Lead — needs to hand auditors and stakeholders a fixed evidence record without giving them system access.
- **Secondary personas (if any)**: External auditors / stakeholders who receive the snapshot; they have no Bunkai login.
- **Business value proposition**: Enables compliance and audit workflows without granting system-level access to external parties. The "read-only, point-in-time" guarantee provides a legally defensible evidence record.
- **KPI(s) influenced**: Audit pass rate, time-to-evidence-delivery, stakeholder trust in traceability.
- **User journey position**: Capstone read-side feature over the test-execution layer — the export is the terminal step of the traceability chain (read the assembled chain → export as snapshot).

### Technical context
- **Frontend**: No existing export UI affordances in the codebase. A new export button/trigger on the chain view page, plus a snapshot retrieval/download view.
- **Backend**: No existing export/download endpoints. Requires a new route (e.g. `GET /api/v1/user-stories/{id}/snapshots` or `POST /api/v1/user-stories/{id}/export`). No PDF/document-generation libraries are installed in the project dependencies (confirmed in the previous shift-left cycle).
- **External services**: None identified. The export artifact lives inside Bunkai's own infrastructure (file storage or DB).
- **Integration points specific to this Story**: Chain assembly output from BK-45 is the input dataset for BK-50. Every entity BK-45 surfaces must be serializable into the snapshot format.

### Story complexity
| Axis | Rating | Why |
|------|--------|-----|
| Business logic | Medium | The export logic itself is straightforward (serialize chain → produce artifact). But the artifact-format decision (file vs DB-copy vs hybrid) adds significant complexity depending on the choice. |
| Integration | Medium | Depends on BK-45's chain-assembly contract which is still undefined. Additionally, RLS/tenant-scoping must be re-verified at export time. |
| Data validation | High | Snapshot immutability (AC2) is the hardest concern: the system must preserve the chain exactly as-of export time despite later mutations to the live chain. |
| UI | Low-Medium | Export trigger button + snapshot retrieval view. Most complexity is backend-oriented. |

**Estimated test effort**: Medium-High (6-10 outlines depending on PO answers to format and access-model questions).

### Epic-level inheritance
- Risks restated at Story level: the Epic (BK-44) explicitly documents that this is **"a read-side capstone over the test-execution layer... scheduled to be implemented once those land."** BK-50 inherits this sequencing constraint.
- Integration points inherited: US → AC → ATC → Test → Run → Defect chain (BK-45's responsibility).
- PO/Dev answers already given at epic level: none — BK-45 still has open questions about the chain response shape.
- Test strategy inherited: API-level testing for the export endpoint + UI-level for the trigger/retrieval affordances.

---

## Phase 2 — Story Quality Analysis

### Ambiguities
| # | Location in Story | Question for PO/Dev | Impact on testing | Suggested clarification |
|---|-------------------|---------------------|-------------------|------------------------|
| 1 | AC1 "a read-only snapshot is produced" | What is the artifact FORMAT — downloadable file (PDF/JSON/HTML), a frozen in-app view, or a DB-copy with a retrieval view? | Cannot design assertions for the output; determines whether tests are file-format assertions or API-response assertions | Specify artifact format explicitly |
| 2 | AC2 "snapshot reflects the moment of export" | What MECHANISM guarantees this — a deep copy of all chain data, or a generated static document inherently frozen? | Deep copy requires DB-state tests; static document requires generation-fidelity tests. Different test designs. | Choose deep-copy vs static-generation explicitly |
| 3 | ACs do not mention WHO can trigger export | Is export permission tied to the "QA Lead" role, or any authenticated user with story read access? | Authorization test scope undefined | Add AC for export permission gate |
| 4 | Story says "without giving them system access" but no AC defines retrieval path | How does an external auditor ACCESS the snapshot — a scoped share link, a downloadable file, both? | Determines whether we need unauthenticated-access tests alongside authenticated ones | Define external-access model |

### Gaps (missing info)
| # | Type | Why critical | What to add | Risk if omitted |
|---|------|--------------|-------------|-----------------|
| 1 | AC | No AC defines snapshot storage/retention policy | Add AC defining whether snapshots are retrievable indefinitely, time-limited, or one-time download | Evidence lost after export |
| 2 | AC | No error path for export when the chain assembly is unavailable (BK-45 down or incomplete) | Add Negative AC for export failure on incomplete chain | System shows confusing error or hangs |
| 3 | AC | No AC for very large chains (synchronous vs async export) | Add AC defining export timeout/async behavior for large chains | UI request times out on large evidence chains |
| 4 | Security | No AC is explicit about whether the export endpoint re-verifies RLS/workspace scoping | Add AC that export enforces the same tenant isolation as the chain read endpoint | Cross-workspace data leak via export |
| 5 | Business rule | What constitutes the "assembled chain" at export time — does it include ALL evidence (ACs, ATCs, Tests, Runs, Defects) or a subset? | Define the exact entity set included in an export | Snapshot missing entities the QA Lead expected |

### Edge cases not in Story
| # | Scenario | Expected behavior (best guess) | Criticality | Action |
|---|----------|-------------------------------|-------------|--------|
| 1 | Exporting the same story twice in quick succession produces two independent snapshots | Both snapshots are independently retrievable, no deduplication | Low | Test only — don't add AC |
| 2 | Snapshot remains accessible after its source story is deleted/archived | Snapshot is a standalone artifact; deletion of source story does NOT affect the snapshot | High | Add to AC (PO confirm) |
| 3 | Export attempted on a user story the requesting user has no read access to | 403/404 returned; no snapshot created; no chain data disclosed | High | Add to AC (PO confirm) — security-relevant |
| 4 | A defect linked in the snapshot is later merged/closed in the live system | Snapshot shows the original link as it was at export time; unchanged by later state changes | Medium | Add to AC (PO confirm) — direct test of immutability promise |
| 5 | Export format compatibility across future schema changes to the chain | Legacy snapshots remain viewable/retrievable in their original format even if the chain schema evolves | Low | Test only — don't add AC |
| 6 | Snapshot retrieval via a direct/guessed link by a user with no workspace access | Access is rejected (403/404); no snapshot data is leaked | High | Add to AC (PO confirm) — especially important given external-auditor audience |

### Contradictions
No contradictions found. The 3 ACs are internally consistent and align with the user story description.

### Testability validation
**Verdict**: Partial

Issues:
- **Missing error messages**: No AC defines what error/status is shown on export failure (chain unavailable, permission denied, timeout, storage error).
- **No performance/capacity criteria**: "Very large chain" behavior is undefined — what happens with 500+ tests in the chain?
- **Depends on BK-45**: The chain assembly doesn't exist yet. Cannot test the export input until the chain output contract is stable.
- **Access model undefined**: Cannot test external-auditor retrieval path until PO defines the share-link vs download vs login-gate model.

---

## Phase 3 — Refined Acceptance Criteria

### Original AC1 — Export an evidence chain

#### Scenario 1.1: Should produce a read-only snapshot containing the full assembled chain when the QA Lead exports a populated user story (Type: Positive, Priority: High)
- ***NEEDS PO/DEV CONFIRMATION***: artifact format inferred — confirm before sprint planning
- **Given**: a user story with an assembled evidence chain (AC → ATC → Test → Run → Defect, mixed pass/fail)
- **When**: the QA Lead triggers the export action
- **Then**: a snapshot artifact is produced containing every chain entity and field visible on screen at the time of export
  - API: 201 Created with snapshot location header OR 200 with artifact body
  - Artifact: contains all chain entities with their full fields (IDs, descriptions, statuses, dates, linked relationships preserved)
  - Snapshot is retrievable via its own URL/ID

#### Scenario 1.2: Should reject export of a user story the requesting user has no read access to (Type: Negative, Priority: High)
- ***NEEDS PO/DEV CONFIRMATION***: behavior inferred — confirm before sprint planning
- **Given**: a user story belonging to a different workspace than the requesting user's
- **When**: the user attempts to export it (e.g. via a crafted story ID)
- **Then**: the export is rejected with 403/404, no snapshot is created, no chain data is disclosed
  - API: 403 Forbidden (or 404 to avoid existence disclosure)
  - No record in the snapshots table (if stored in DB)
  - No file artifact created (if stored as file)

### Original AC2 — Snapshot reflects the moment of export

#### Scenario 2.1: Should preserve the chain's state at export time when the live chain changes afterward (Type: Positive, Priority: Critical)
- ***NEEDS PO/DEV CONFIRMATION***: mechanism inferred — confirm before sprint planning
- **Given**: a user story exported at time T0 with a specific chain state (e.g. AC-1 → Test-A → Run "pass")
- **When**: after export, the live chain changes (e.g. a new Run is added with verdict "fail", or an existing Defect is closed)
- **When**: the QA Lead later opens the previously exported snapshot
- **Then**: the snapshot still displays the chain exactly as it was at T0, unaffected by the later changes
  - Snapshot shows the Run "pass" verdict, not the later updates
  - Snapshot shows the Defect in its original state
  - The live chain and the snapshot differ (proof of independence)

#### Scenario 2.2: Should produce a unique, independent snapshot for concurrent exports of the same story (Type: Positive, Priority: Medium)
- ***NEEDS PO/DEV CONFIRMATION***: behavior inferred — confirm before sprint planning
- **Given**: a user story with a stable chain state
- **When**: the QA Lead triggers export at time T0, and the same story is exported again at T1 (seconds apart)
- **Then**: two independent snapshots exist; each reflects the chain state at its export moment; retrieving one does NOT return the other.

### Original AC3 — Export an empty chain

#### Scenario 3.1: Should produce a snapshot stating the story had no coverage when exporting a story with zero chain entities (Type: Negative/Edge, Priority: High)
- ***NEEDS PO/DEV CONFIRMATION***: exact copy inferred — confirm before sprint planning
- **Given**: a user story with no ACs, ATCs, Tests, Runs, or Defects (zero coverage)
- **When**: the QA Lead exports it
- **Then**: a snapshot artifact is produced stating the story had no coverage as of the export timestamp (exact copy TBD by PO)
  - Snapshot header/status: clearly states "No coverage" or equivalent
  - Export timestamp is displayed
  - No error or crash occurs despite the empty input

### New scenarios surfaced from Phase 2 edge cases — NEEDS PO/DEV CONFIRMATION

#### Scenario E1: Should keep a previously exported snapshot accessible after its source user story is deleted/archived (Type: Edge, Priority: High)
- ***NEEDS PO/DEV CONFIRMATION***: behavior inferred — confirm before sprint planning
- **Given**: a snapshot exported from user story X at time T0
- **When**: user story X is later deleted or archived
- **Then**: the snapshot remains retrievable and fully viewable as a standalone artifact, unaffected by the source story's deletion
  - Snapshot URL/ID still returns the artifact
  - All chain data is still readable within the snapshot
  - No "source not found" error is displayed

#### Scenario E2: Should reject unauthenticated/unauthorized attempt to view a snapshot via direct link (Type: Negative, Priority: High)
- ***NEEDS PO/DEV CONFIRMATION***: behavior inferred — confirm before sprint planning
- **Given**: a snapshot exists, and its retrieval URL/ID is known or guessed by a user without access to the original story's workspace
- **When**: that user attempts to retrieve the snapshot
- **Then**: access is rejected (403/404), no snapshot data is disclosed

#### Scenario E3: Should display a clear error when export is attempted while the chain assembly is unavailable (Type: Negative, Priority: Medium)
- **Given**: a user story whose chain assembly endpoint returns 500 or times out
- **When**: the QA Lead attempts to export the story
- **Then**: a clear error message is displayed (e.g. "Chain assembly unavailable. Please try again later.")
  - No partial or corrupted snapshot is created
  - The export action can be retried

---

## Phase 4 — Test Outlines (DRAFT — outline names only)

### Coverage estimate
| Type | Count | Notes |
|------|-------|-------|
| Positive | 3 | Full chain export, concurrent exports, independent snapshot identity |
| Negative | 4 | Unauthorized story, unauthorized snapshot view, chain unavailable, empty story export |
| Boundary | 2 | Very large chain (performance), chain changing during export (race condition) |
| Integration | 2 | Chain assembly dependency (BK-45), RLS/tenant-scoping enforcement |
| **Total** | **11** | May increase after PO answers format and access-model questions |

**Rationale**: The Story touches 3 distinct ACs, each with multiple execution paths (happy + error). The immutability requirement (AC2) is the most complex — it requires state before/after mutation tests. Auth/security adds the negative and boundary outlines. Integration with BK-45 adds the dependency-contract outlines.

### Outline list (NAMES ONLY)

#### Positive
- **Should produce a downloadable snapshot with full chain data when a QA Lead exports a populated story** — Pre: story with assembled chain (AC → ATC → Test → Run → Defect). Expected: 201/200 with artifact containing all chain entities.
- **Should produce independent snapshots when the same story is exported twice in succession** — Pre: stable chain state. Expected: two separate snapshots with distinct IDs, both retrievable.
- **Should display "no coverage" message when exporting a story with zero chain entities** — Pre: story with no ACs/ATCs/Tests/Runs/Defects. Expected: snapshot stating "No coverage" with export timestamp.

#### Negative
- **Should reject export of a story the requesting user has no read access to** — Pre: story in different workspace. Expected: 403/404, no snapshot created.
- **Should reject snapshot view attempt by an unauthenticated user via direct link** — Pre: existing snapshot. Expected: 403/404 or login redirect, no data leaked.
- **Should display clear error when export is triggered while chain assembly is unavailable** — Pre: BK-45 endpoint returns 500. Expected: user-visible error, no partial snapshot.
- **Should reject export for a deleted/archived story** — Pre: story was deleted. Expected: 404, clear error message.

#### Boundary
- **Should handle export of a very large chain (500+ entities) without timeout** — Pre: story with 500+ chain entities. Expected: export completes within defined timeout (TBD), artifact is complete.
- **Should handle chain state changing mid-export (race condition)** — Pre: chain data being mutated during export. Expected: snapshot reflects either pre-mutation or post-mutation state, never a partial/corrupt state.

#### Integration
- **Should verify export enforces same RLS/tenant-scoping rules as live chain view** — Pre: story visible to user A but not user B. Expected: user B's export rejected with 403/404.
- **Should handle BK-45 chain assembly response format changes gracefully** — Pre: BK-45 returns new/unknown fields. Expected: snapshot serializes known fields, logs unknown fields, does not crash.

> **NOT included here** (deferred to in-sprint planning by `/sprint-testing` Stage 1): parametrization tables, per-outline test-data JSON, numbered test steps, Faker generation strategies.

---

## Phase 5 — Edge Cases (DRAFT)

| # | Edge case | In original Story? | Criticality | Action |
|---|-----------|-------------------|-------------|--------|
| 1 | Exporting the same story twice in quick succession | No | Low | Test only — don't add AC |
| 2 | Snapshot remains accessible after source story deleted/archived | No | High | Add to AC (PO confirm) |
| 3 | Export attempted on a story outside requester's workspace (crafted ID) | No | High | Add to AC (PO confirm) — security-relevant |
| 4 | A defect linked in the snapshot is later merged/closed in live system | No | Medium | Add to AC (PO confirm) — tests immutability |
| 5 | Snapshot format compatibility across future chain schema changes | No | Low | Test only — don't add AC |
| 6 | Snapshot retrieval via direct/guessed link by user with no workspace access | No | High | Add to AC (PO confirm) — security |
| 7 | Story chain assembly endpoint is down/500s at export time | No | Medium | Add to AC or test only |
| 8 | Chain state mutates during export (race between read and serialize) | No | Medium | Test only |
| 9 | Exporting a story with 500+ chain entities (large chain) | No | Medium | Add to AC or test only |
| 10 | Snapshot content depends on the user who originally exported (different RLS visibility) | No | Medium | Test only |

> Test-data generation strategy + Faker recipes are NOT defined here. They land in `/sprint-testing` Stage 1 when the feature exists.

---

## Story Quality Assessment

**Verdict**: Significant Issues — but this is expected for a story whose upstream dependency (BK-45) is still in Estimation.

**Key findings**:
1. The 3 ACs are clear in INTENT but leave the single most consequential implementation decision completely open: HOW is "read-only snapshot" realized — a downloadable file, a frozen in-app view, or a DB-copy record? This decision changes the data model, the access-control model, and nearly every outline.
2. The dominant issue is the complete non-existence of the feature being exported: BK-50 exports a chain that does not yet exist (BK-45).
3. The "read-only" framing and the Story's explicit goal of external-auditor access implies an access-control surface that none of the 3 ACs currently address — this is a security-relevant gap.

---

## Critical Questions for PO

> These BLOCK sprint planning until answered.

1. **What artifact format does "export" produce — a downloadable file (PDF/JSON/HTML), or an in-app read-only view still gated by authentication?**
   - **Context**: The user story says "without giving them system access," suggesting a standalone file or unauthenticated share link rather than a logged-in view.
   - **Impact if unanswered**: Nearly every test outline changes shape depending on this answer.
   - **Suggested answer (if you have one)**: Start with JSON export (same format as the BK-45 chain response) — it is implementable without new file-generation libraries and covers the core export need.

2. **What mechanism guarantees the snapshot reflects the moment of export (AC2) — a deep copy or a generated static document?**
   - **Context**: A deep copy stores chain data separately; a static document is inherently frozen.
   - **Impact if unanswered**: The immutability test design depends entirely on this mechanism.
   - **Suggested answer**: Deep copy in a snapshots table — it is queryable, individually retrievable, and survives source-story deletion.

3. **Who is allowed to trigger an export, and what access model governs who can later retrieve/view an already-exported snapshot?**
   - **Context**: The Story explicitly targets external auditors with no system login.
   - **Impact if unanswered**: Security-relevant — cannot add access-control tests without the access model.
   - **Suggested answer**: For v1, tie export permission to "read access to the user story" and make snapshots retrievable only by the same user who created them. Extend for external access in a follow-up.

---

## Technical Questions for Dev

> These do not block PO but block implementation.

1. **Will the export run synchronously or asynchronously?**
   - **Context**: Phase 2 gap; the Epic's risk map flags N+1/performance risk at the chain-assembly layer.
   - **Testing impact**: Determines whether a large-chain export is tested as a simple synchronous-response assertion or requires polling/notification-flow test design.

2. **If the snapshot is a DB-copy rather than a static file, what is the storage/retention policy?**
   - **Context**: No AC addresses retention.
   - **Testing impact**: Determines whether "list past exports" and expiry/cleanup test outlines are in scope at all for v1.

3. **Does the export endpoint independently re-verify RLS scoping at generation time, or does it trust the caller's already-authenticated session context?**
   - **Context**: An artifact that leaves the system is a higher-stakes surface for a missed RLS check than an in-app read.
   - **Testing impact**: Determines whether the export endpoint needs its own dedicated tenant-isolation test, separate from BK-45's.

---

## Suggested Story Improvements

| # | Current state | Suggested change | Benefit |
|---|---------------|------------------|---------|
| 1 | "a read-only snapshot is produced" (AC1) | "exporting produces a [file format TBD] containing the full chain, retrievable without requiring the recipient to log into Bunkai" | Removes format ambiguity and makes the external-access promise testable |
| 2 | "the snapshot still shows the evidence as it was at export time" (AC2) | "the snapshot is generated as a [static document / deep copy — PO to choose] that is structurally independent of the live chain" | Makes immutability concrete enough to test against deletion/mutation edge cases |
| 3 | No AC on who can export or who can later view | Add AC: "Only users with read access to the user story can trigger export" | Prevents unintentionally open access surface |
| 4 | "the snapshot states the story had no coverage at export time" (AC3) | "the snapshot states the story had no coverage at export time, using the copy: '[exact wording TBD]'" | Makes the AC3 assertion deterministic |

---

## Data feasibility flags

***DATA-FEASIBILITY-RISK***: confirmed and concrete.

- **Entity / fixture missing**: There is no queryable data structure to export. BK-45's own refinement confirms zero implementation of `tests`, `test*runs`/`run*results`, or `defects`/`bugs` tables across all reviewed migrations, and no `GET /api/v1/user-stories/{id}/traceability` endpoint exists. BK-50 has nothing to export.
- **API contract gap**: An export capability needs a stable chain-assembly response shape to serialize. That shape is still under PO/Dev negotiation in BK-45's open questions.
- **Required pre-work**: (1) BK-45 must reach at least a stable, documented chain-assembly contract before BK-50 can be implemented or meaningfully estimated. (2) PO must decide the export mechanism and external-access model BEFORE estimation.

---

## Recommended testing strategy

### Pre-implementation
- Do not write parametrized test-data or numbered test steps yet — defer to in-sprint planning once BK-45's chain contract is stable AND the export mechanism is decided.
- Track BK-45's status and the PO's answer to Critical Question 2 (persistence mechanism) — that single decision reshapes most of Phase 4's outlines.
- Resolve the 3 Critical PO Questions and 3 Dev questions before any SP estimation session.

### During implementation
- Verify the export endpoint independently enforces RLS/tenant scoping (Tech Q3) early — an artifact that leaves the system is a higher-stakes surface for a missed isolation check.
- Verify the chosen immutability mechanism (static file vs DB-copy) actually holds under the realistic mutation scenarios (source story deleted; linked defect later merged) — these are the scenarios most likely to silently break the "moment of export" promise.

### Post-implementation (in-sprint by /sprint-testing)
- Expand the 11 DRAFT outlines into full parametrized test cases with concrete chain shapes and mutation timelines.
- Add the deferred edge cases (Phase 5, especially #2, #3, #4, #6) as formal ACs or test-only cases per PO's confirmation.
- Design the external-access-model test suite (tokenized link expiry, revocation, no-login retrieval) once Critical Question 3 is answered.

---

## Risks & mitigation

| # | Risk | Likelihood | Impact | Mitigated by which outlines |
|---|------|-----------|--------|-----------------------------|
| 1 | BK-50 estimated/scheduled before BK-45 ships, committing sprint capacity to an unbuildable Story | High | High | N/A — mitigated by sequencing recommendation |
| 2 | "Read-only snapshot" implemented as a live, re-fetched view rather than a true point-in-time copy, silently breaking AC2 | Medium | Critical | Outline "Should preserve chain state after live changes" |
| 3 | Exported artifact leaks data across workspace boundaries because the export endpoint trusts session context instead of re-verifying RLS | Low | Critical | Outline "Should enforce RLS/tenant-scoping" + Critical Question 3 |
| 4 | Snapshot becomes inaccessible once its source story is deleted/archived, defeating the "fixed record" promise | Medium | High | Outline "Should keep snapshot accessible after source story deletion" |
| 5 | No export permission gate implemented, allowing any authenticated user (not just QA Lead) to export sensitive evidence chains | Medium | Medium | Outline "Should reject unauthorized export" + Critical Question 3 |

---

## Next steps

- [ ] PO answers Critical Questions before sprint planning
- [ ] Dev answers Technical Questions before estimation
- [ ] Story enters sprint at status Ready For Dev once estimated
- [ ] When Story reaches Ready For QA, `/sprint-testing` will short-circuit refinement (label `shift-left-reviewed` detected)
- [ ] ***BLOCKER***: Do not estimate or schedule BK-50 ahead of BK-45 reaching a stable chain-assembly contract
- [ ] ***BLOCKER***: Do not estimate BK-50 until PO/Dev decide the export-artifact format and the external-access model (Critical Questions 1-3)
