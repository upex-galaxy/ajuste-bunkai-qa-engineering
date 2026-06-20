# Business Model — Bunkai (QA Context)

> Product: **Bunkai** (分解) — open-core Test Management System.
> Status: pre-MVP. Cloud-validation on Next.js + Supabase, self-hosted Docker Compose planned Phase 2.
> Source: target `.context/business/business-model.md` (canonical). This is a QA-focused adaptation.

---

## Problem Statement

Existing TMSs (Xray, Zephyr, TestRail, qTest) are document vaults — they store test cases as monolithic blobs, enforce no structural traceability, and delegate bugs to Jira. The result: duplicated steps, broken trazabilidad, maintenance rot, and reports that lie.

**Bunkai's thesis**: a data model that enforces structural traceability (ATC → US → AC → Module → Bug) makes good QA the path of least resistance.

## Business Model Canvas

| Block | Detail | QA Relevance |
|---|---|---|
| **Customer Segments** | Indie QA engineers, mid-market orgs, regulated-industry enterprises, QA training audience | Testing must cover all persona workflows (Elena, Mateo, Sara, Karim) |
| **Value Propositions** | ATC reuse, structural traceability, 3 execution modes (manual/agentic/automated), native bugs, API-first, open-source | Core features to test: ATC chaining, traceability invariants, Run state machine, Bug lifecycle |
| **Channels** | GitHub-led distribution, content-led inbound, conference talks, UPEX Galaxy course | Test signup flow, OAuth, workspace onboarding |
| **Revenue Streams** | Open-core: Community (free), Cloud (per-seat), Enterprise (annual license) | Edition-gated features must be tested per tier |
| **Key Activities** | Building/maintaining open-source core, operating Cloud infra, docs, community | Regression suite for core features; smoke for Cloud-specific features |
| **Key Partners** | Vercel, Supabase, Cloudflare R2, Jira (bidirectional sync) | Integration testing for Jira sync, deploy verification on Vercel |
| **Key Resources** | Open-source codebase, founder QA reputation, KATA methodology, brand | Codebase is the test surface — KATA alignment is a differentiator |

## Key Metrics (MVP)

| Metric | Target | Testing Implication |
|---|---|---|
| Activation (% workspaces with ≥1 Module + ≥1 ATC + ≥1 Test + ≥1 Run within 24h) | ≥60% | Test the full "first run" user journey end-to-end |
| ATC reuse ratio (tests-per-ATC) at day 30 | <1.0 | Verify ATC chaining works across Tests; edits propagate |
| Structural correctness (% ATCs anchored to US + AC) | 100% | Data-model invariant — test that unanchored ATCs are rejected |
| Three-mode adoption (≥2 modes within 30 days) | TBD | API-first design means all UI actions have REST endpoints |

## Discovery Gaps

- [ ] Actual pricing tiers not finalized (target TBD)
- [ ] Self-hosted Docker Compose edition not implemented yet
- [ ] Agentic execution protocol not built (Phase 2)
- [ ] CI/CD adapter imports not built (Phase 2)

## Sources Used

| Claim | Source |
|---|---|
| Problem statement | target `.context/business/business-model.md` §Problem Statement |
| Customer segments | target `.context/business/business-model.md` §1 |
| Value propositions | target `.context/business/business-model.md` §2 |
| Revenue model | target `.context/business/business-model.md` §5 |
| MVP metrics | target `.context/PRD/executive-summary.md` §3 |
| Tech stack | target `.agents/project.yaml` |
