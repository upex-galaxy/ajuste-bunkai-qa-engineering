# `.context/PBI/` — Product Backlog Items (QA Context)

> Backlog access recipe for Bunkai (project key: BK).
> Issue tracker: Jira Cloud → resolved via `[ISSUE_TRACKER_TOOL]` (acli).

---

## Project

- **Project Key**: BK (Bunkai)
- **Jira URL**: `https://upexgalaxy67.atlassian.net/`
- **TMS Provider**: Xray (Modality A)

## Authentication

```bash
# acli authentication
acli auth login --site upexgalaxy67.atlassian.net
```

Required env vars (already in `.env`):
```
ATLASSIAN_URL=https://upexgalaxy67.atlassian.net/
ATLASSIAN_EMAIL=<email>
ATLASSIAN_API_TOKEN=<api-token>
```

## Common Queries

```jql
# Current sprint
project = BK AND sprint in openSprints() ORDER BY priority DESC

# My open issues
project = BK AND assignee = currentUser() AND status NOT IN (Closed, Done, Resolved)

# Backlog (unscheduled)
project = BK AND sprint IS EMPTY AND status = 'Backlog' ORDER BY priority DESC

# Bugs by module
project = BK AND issuetype = Bug AND status NOT IN (Closed) ORDER BY created DESC

# Stories ready for QA
project = BK AND status = 'Ready For QA' ORDER BY priority DESC
```

## PBI Layout

```
.context/PBI/
├── README.md                       # This file
├── templates/
│   ├── user-story.md               # Template for user story testing
│   ├── bug-report.md               # Template for bug reporting
│   └── test-plan.md                # Template for test plans
```

Per-ticket PBI folders are synced from Jira via `bun run jira:sync-issues` during `/sprint-testing`.
Jira is the source of truth — local PBI files are read-only caches.

## Template Guides

- `templates/user-story.md` — for documenting Story-level test analysis during `/sprint-testing` Stage 1
- `templates/bug-report.md` — for bug reproduction documentation
- `templates/test-plan.md` — for feature/module-level test plans (ATP)
