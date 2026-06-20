# Executive Summary — Bunkai (QA Context)

> **Product**: Bunkai (分解) — open-core Test Management System.
> **Stack**: Next.js 15 (App Router) + Supabase + Vercel.
> **Source**: target `.context/PRD/executive-summary.md` (canonical). This is a QA-focused adaptation.

---

## Problem Statement

Existing TMSs (Xray, Zephyr, TestRail, qTest) treat test cases as monolithic blobs. Steps are duplicated across tests, traceability between US → AC → ATC → Run → Bug is optional, reports lie, and bugs are delegated to Jira. Regulated industries (fintech, healthtech, legaltech) cannot use SaaS TMSs over data sovereignty.

**Bunkai's solution**: a data model where every ATC must be anchored to a US + AC, tests are ordered ATC chains (not free-form steps), bugs live natively inside the test cycle, and the same data model serves manual, agentic, and automated execution.

## Core Capabilities (MVP)

| Feature | Problem Addressed | Source |
|---|---|---|
| ATC Library (atomic, reusable, anchored) | Duplication, maintenance rot | `functional-specs.md` BK-011–016 |
| Tests as ATC chains | Monolithic test cases | `functional-specs.md` BK-017–020 |
| Manual execution with step-by-step runner | Executable test evidence | `functional-specs.md` BK-021–026 |
| Native bug management | Bug context loss, Jira delegation | `functional-specs.md` BK-027–030 |
| Tree view + Table view | Navigation, inventory | `mvp-scope.md` EPIC-BK-008 |
| REST API + OpenAPI | AI operators, CI/CD foundation | `functional-specs.md` BK-036–039 |
| Jira import (one-way) | Migration from existing TMS | `functional-specs.md` BK-009 |
| RBAC (owner/admin/member/viewer) | Multi-tenant access control | `functional-specs.md` BK-003 |

## Key Differentiators

1. **ATC as first-class entity** — no competitor enforces atomic reusable components
2. **Three execution modes, one data model** — manual, agentic, automated produce comparable runs
3. **Structural traceability, not optional** — data model prevents orphan tests
4. **Open-source + self-hostable** — data sovereignty for regulated industries
5. **API-first for AI operators** — every UI action has a REST endpoint

## Success Metrics

| Metric | Type | Target | Testing Implication |
|---|---|---|---|
| Activation (Module+ATC+Test+Run in 24h) | Adoption | ≥60% | Test "first run" journey end-to-end |
| ATC reuse ratio | Engagement | <1.0 at D30 | Verify ATC chaining + propagation |
| Structural correctness | Quality | 100% | Test data-model invariants (unanchored ATC rejected) |

## Target Users

| Persona | Role | Primary Need |
|---|---|---|
| Elena Vargas | Senior QA Engineer | ATC reuse, traceability, maintenance efficiency |
| Mateo Silva | QA Lead/Manager | Coverage dashboards, heatmaps, audit trail |
| Sara Iglesias | Full-Stack Developer | "Is my feature tested?" visibility |
| Karim | AI Agent (non-human) | Deterministic API, idempotency, streaming results |

Detailed personas in `user-personas.md`.

## Scope (MVP)

**In scope**: 7 epics (Tenancy, Project/Module hierarchy, US+AC, ATC library, Test chains, Manual execution, Bugs) + Views + API/CLI.

**Out of scope**: Mind-map view (Phase 2), 3D toggle (Phase 3), pgvector semantic search (Phase 2), agentic execution protocol (Phase 2), CI/CD import adapters (Phase 2), SSO/SAML (Phase 3), Docker Compose self-hosted (Phase 2).

## QA Relevance

This is a **Test Management System**. Its core features ARE testing constructs (ATCs, Runs, Bugs). Testing Bunkai requires:
- Verifying the data-model invariants (no orphan ATCs, no unanchored Runs)
- Testing the state machines (Run lifecycle, Bug lifecycle)
- Ensuring the API surface matches the OpenAPI contract
- Validating the three execution modes produce compatible data shapes
