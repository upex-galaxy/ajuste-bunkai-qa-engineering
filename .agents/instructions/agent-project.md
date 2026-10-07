---
id: project
title: 'Project-specific instructions'
load_when: 'anything specific to this project: its own conventions, guardrails, environments or vocabulary not covered by another section'
triggers: []
paths: []
---

# Project-specific instructions

> Project-owned overlay. The boilerplate delivers this file once and never overwrites it; every other file in this folder is synced from upstream. Write this project's own rules here, one heading per topic, instead of editing `AGENTS.md` or a synced section. Knowledge about the system under test belongs in a project context skill (`<aspect>-context`), not here.

Add `triggers:` regex sources to the frontmatter above when a rule here should be routed by keyword (the hook reads them), and keep every `NEVER` / `MUST` line reachable: cite `Rule #N`, binding: `/<skill>`, or enforced: `bun run <script>` (checked by `bun run instructions:check`).

## Project context skills

One row per skill this project authored: the `<aspect>-context` skills `project-context` mode `context-skill` creates, and any other skill the project added. The skill router in `agent-skills-and-mcps.md` is synced, so `bun run up` overwrites a row written there; this file is never overwritten. Add each skill's trigger phrases to `triggers:` above as regex sources too, so the hook routes this file when a prompt names them. `bun run instructions:check` fails a row whose `.agents/skills/<slug>/SKILL.md` does not exist.

| Skill | Trigger | Purpose |
|---|---|---|

## Project Assessment (Phase 1)

Assessment Date: 2026-06-18

### Testing Maturity: 0/4
- Current state: None — no test files found in target repo (`upex-bunkai-tms`)
- Test files: 0
- Frameworks: None detected (pre-MVP project)
- Coverage: N/A
- Notes: Target is a pre-production project. All QA testing infra lives in this repo (ajuste-bunkai-qa-engineering), not in the target. Greenfield automation opportunity.

### Documentation State: Good
- README: yes (comprehensive, boilerplate onboarding)
- API docs: `/api` contracts defined in `.context/SRS/api-contracts.yaml`
- Architecture: yes (C4 diagrams, ERD, component specs)
- Setup guide: yes (README + INSTALLER.md)

### Code Quality
- [x] ESLint: configured (`@antfu/eslint-config`)
- [x] Prettier: configured
- [x] TypeScript: configured (strict mode expected)
- [ ] Pre-commit hooks: Husky configured but only runs lint-staged

### CI/CD Maturity: Basic
- No GitHub Actions workflows detected in target repo
- Build/typecheck/lint run locally via `bun run repo:check`
- Vercel auto-deploys on push (staging branch + main)

### Identified Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| No CI/CD pipelines | MEDIUM | Set up GH Actions during /adapt-framework or immediately after |
| No OpenAPI spec served at runtime | MEDIUM | Target has `api-contracts.yaml` in SRS — confirm live endpoint or generate via `bun run api:sync` |
| DBHub credentials not configured | MEDIUM | Populate `.env` DBHUB_* vars before [DB_TOOL] usage |
| Pre-MVP project — features may shift | LOW | Track via PBI sync; re-run `/business-data-map` as needed |

### Phase Prioritization

- Phase 1: Normal — target has excellent docs, minimal reverse-engineering needed
- Phase 2: Normal — adapt existing PRD/SRS into QA context
- Phase 3: Normal — infrastructure gap analysis is the main value-add
- Phase 4: Normal — standard PBI mapping

### Blockers
- [ ] No CI pipelines in target — set up at least smoke tests before regression testing phase

---

*AI persistent memory. Update when behaviors / skills / rules change.*
