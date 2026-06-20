# Project Configuration

> Project: Bunkai
> Generated: 2026-06-18

## Repositories

| Repository | URL | Branch | Purpose |
|---|---|---|---|
| upex-bunkai-tms | `../upex-bunkai-tms` | main | Monorepo — Next.js app (FE + API routes) |

## Tech Stack

### Frontend
- Framework: Next.js 15 (App Router, RSC, typedRoutes)
- Language: TypeScript 5.9
- Styling: Tailwind CSS 3.4 + shadcn/ui (Radix UI primitives)
- State: React Server Components + TanStack React Table (client views)
- Editor: Monaco Editor (ATC authoring)
- Component libs: lucide-react, cmdk (command palette), sonner (toasts), class-variance-authority

### Backend
- Framework: Next.js Route Handlers (`app/api/v1/...`)
- Language: TypeScript
- ORM: Supabase JS client (raw SQL via RPC)
- Auth: Supabase Auth (JWT, OAuth GitHub/Google, magic link)
- Validation: Zod schemas (server + client)

### Database
- Type: PostgreSQL 16
- Provider: Supabase
- Features: Row Level Security, Realtime (logical replication), tsvector search
- Access: Supabase JS client (`@supabase/supabase-js`, `@supabase/ssr`)

### Infrastructure
- Cloud: Vercel (frontend + serverless functions)
- Object Storage: Cloudflare R2 (run evidence blobs)
- Monitoring: Sentry
- Analytics: PostHog
- CI/CD: Not yet configured (pre-MVP)

## Environments

| Environment | URL | Purpose | Auth |
|---|---|---|---|
| Local | `http://localhost:3000` | Development | Supabase local |
| Staging | `https://staging-upexbunkai.vercel.app` | Pre-prod testing | Supabase staging |
| Production | `https://upexbunkai.vercel.app` / `https://bunkai.io` | Live | Supabase production |

## Tools and Access

- Issue tracker: Jira — resolved via [ISSUE_TRACKER_TOOL] (acli)
- Project key: BK
- Database: resolved via [DB_TOOL] (DBHub MCP)
- Docs: in-repo `.context/` (no external wiki)

## Access Checklist

- [x] Repository read access
- [ ] Database access (MCP or direct — env vars not fully populated)
- [x] Issue tracker access (Jira configured: ATLASSIAN_URL, ATLASSIAN_EMAIL, ATLASSIAN_API_TOKEN)
- [x] Staging environment reachable (staging-upexbunkai.vercel.app)
- [ ] CI/CD visibility (no workflows configured yet)

## Discovery Gaps

- [ ] No CI/CD pipelines detected in target repo (< 2026-06-18)
- [ ] OpenAPI spec not confirmed (`api-contracts.yaml` exists in target's SRS, but live endpoint may differ)
- [ ] Database MCP credentials not fully populated in `.env` (DBHUB_* vars empty)
