# Architecture Specifications — Bunkai (QA Context)

> Source: target `.context/SRS/architecture-specs.md`. C4 + ERD + component specs adapted for QA context.
> Stack: Next.js 15 (App Router) + Supabase (PostgreSQL, Auth, Realtime, Storage) + Vercel + Cloudflare R2.

---

## System Overview

**Pattern**: Next.js App Router with React Server Components. API routes handle backend logic. Supabase provides DB, Auth, Realtime, and Storage. Serverless on Vercel.

### Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15 (App Router, RSC, typedRoutes) |
| UI Components | shadcn/ui (Radix UI primitives) + Tailwind CSS |
| Client State | TanStack React Table |
| Editor | Monaco Editor |
| API | Next.js Route Handlers (`app/api/v1/*`) |
| Validation | Zod (server + client) |
| Database | Supabase PostgreSQL 16 |
| Auth | Supabase Auth (JWT, OAuth GitHub/Google, magic link) |
| Realtime | Supabase Realtime (logical replication) |
| Object Storage | Cloudflare R2 (S3-compatible) |
| Monitoring | Sentry |
| Analytics | PostHog |
| Deploy | Vercel |

## C4 Context Diagram

```mermaid
flowchart LR
  subgraph User
    QA[QA Engineer Browser]
    AGT[AI Agent / CLI]
  end

  subgraph Edge[Vercel Edge]
    NXT[Next.js 15 App\nApp Router + RSC\nAPI routes /api/v1]
  end

  subgraph Data[Supabase project]
    PG[(PostgreSQL 16)]
    AUTH[Supabase Auth\nJWT + OAuth]
    RT[Supabase Realtime]
    STG[Supabase Storage]
  end

  subgraph External
    R2[Cloudflare R2]
    JIRA[Jira REST API]
    SEN[Sentry]
    PH[PostHog]
  end

  QA -- HTTPS --> NXT
  AGT -- HTTPS + Bearer --> NXT
  NXT -- pg + Supabase JS --> PG
  NXT -- session --> AUTH
  NXT --> RT
  RT -- WebSocket --> QA
  NXT -- signed URL --> R2
  NXT -- async webhook --> JIRA
  NXT --> SEN
  NXT --> PH
```

## Component Structure

```mermaid
flowchart TD
  subgraph app
    layout[layout.tsx]
    page[page.tsx]
    subgraph routes
      app_routes["(app)/\nprotected routes"]
      auth_routes["(auth)/\nlogin, signup"]
      api_routes["api/v1/\nREST endpoints"]
      qa_page["qa/\ntestability guide"]
      invites["invites/\naccept flow"]
    end
  end

  subgraph lib
    supabase["supabase/\nclient + server"]
    atcs["atcs/"]
    modules["modules/"]
    runs["runs/"]
    bugs["bugs/"]
  end

  subgraph components
    ui["ui/\nshadcn/ui"]
    tree["tree/\ntree view"]
    table["table/\nTanStack table"]
    editor["editor/\nMonaco"]
  end

  layout --> routes
  routes --> lib
  routes --> components
```

## Database Schema (ERD)

```mermaid
erDiagram
  WORKSPACES ||--o{ WORKSPACE_MEMBERS : has
  WORKSPACES ||--o{ PROJECTS : owns
  WORKSPACES ||--o{ WORKSPACE_INVITES : sends
  WORKSPACES ||--o{ ACCESS_TOKENS : issues
  USERS ||--o{ WORKSPACE_MEMBERS : belongs_to
  USERS ||--o{ ACCESS_TOKENS : owns

  PROJECTS ||--o{ MODULES : contains
  PROJECTS ||--o{ ENVIRONMENTS : configures
  PROJECTS ||--o{ INTEGRATIONS : has
  PROJECTS ||--o{ FEATURE_FLAGS : toggles

  MODULES ||--o{ MODULES : parent
  MODULES ||--o{ USER_STORIES : groups
  USER_STORIES ||--o{ ACCEPTANCE_CRITERIA : has
  USER_STORIES ||--o{ ATCS : referenced_by
  ACCEPTANCE_CRITERIA ||--o{ ATC_ACCEPTANCE_CRITERIA : referenced_by

  ATCS ||--o{ ATC_STEPS : ordered
  ATCS ||--o{ ATC_ASSERTIONS : has
  ATCS ||--o{ ATC_ACCEPTANCE_CRITERIA : satisfies
  ATCS }o--|| MODULES : anchored_to

  TESTS ||--o{ TEST_STEPS : chains
  TEST_STEPS }o--|| ATCS : invokes
  TESTS }o--|| MODULES : optional_anchor

  RUNS }o--|| TESTS : executes
  RUNS ||--o{ RUN_ATCS : produces
  RUN_ATCS }o--|| ATCS : snapshot_of
  RUN_ATCS ||--o{ RUN_STEPS : tracks
  RUN_STEPS }o--|| ATC_STEPS : snapshot_of

  BUGS }o--o| ATCS : optionally_links
  BUGS }o--o| RUNS : optionally_links
  BUGS }o--|| MODULES : anchored_to
```

## Auth Architecture

**AuthN**: Supabase Auth (JWT). Three methods:
- Magic-link email
- GitHub OAuth
- Google OAuth

**AuthZ**: RBAC at Workspace level enforced by Supabase Row Level Security (RLS) policies.

**Session**: 1h access token, 30d refresh token, sliding renewal.

Protected routes: `/projects/*`, `/onboarding/*` (via `middleware.ts` using Supabase SSR client).

## Data Flow — Run Execution

```
User/Agent → POST /runs → Run created (created)
  → POST /run_steps/{step_id}/result → Run progresses (running)
  → ALL steps passed → Run→passed
  → ANY step failed → Run→failed
  → Agent aborts → Run→aborted
  → Bug filed (POST /bugs) linked to Run + ATC + Module
```

## External Services

| Service | Purpose | Integration Point |
|---|---|---|
| Supabase | Database, Auth, Realtime, Storage | Supabase JS client |
| Cloudflare R2 | Blob storage for evidence | S3-compatible signed URLs |
| Jira (optional) | Bidirectional issue sync | REST API webhook |
| Sentry | Error monitoring | Sentry SDK |
| PostHog | Product analytics | PostHog SDK |

## Security Architecture

- AuthZ: RLS policies on every table with `workspace_id`
- Input validation: Zod schemas server + client
- Markdown sanitized via `rehype-sanitize`
- Rate limits: 100 req/min write, 600 req/min read
- CSP: strict, nonces for inline scripts
- Secrets: never in code, `.env` for local, Vercel env vars for prod/staging

## QA Relevance

| Component | Test Approach |
|---|---|
| API routes (`app/api/v1/*`) | Contract testing via REST calls |
| RLS policies | AuthZ bypass attempts (cross-workspace access) |
| Auth middleware | Unauthenticated redirect, token expiry |
| Supabase Realtime | Real-time UI updates during Run |
| Jira sync | Bidirectional sync consistency, conflict resolution |
| Cloudflare R2 | Evidence upload/download, signed URL expiry |

## Discovery Gaps

- [ ] Live database schema not confirmed — migrate from Supabase migrations
- [ ] OpenAPI spec served at `/api/openapi.json` — verify at runtime
- [ ] Real Supabase RLS policies — need to read from `supabase/migrations/`
- [ ] Jira integration not fully implemented (MVP scope)
