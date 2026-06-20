# Backend — Bunkai (QA Context)

> Source: target code analysis (`package.json`, `middleware.ts`, `next.config.ts`, `supabase/` directory, `.agents/project.yaml`).

---

## Runtime

| Attribute | Value |
|---|---|
| Language | TypeScript 5.9 |
| Runtime | Bun 1.0+ (local), Node.js (Vercel serverless) |
| Framework | Next.js 15 (App Router, Route Handlers) |
| Auth | Supabase Auth + Supabase SSR (`@supabase/ssr`) |
| Validation | Zod 4 |
| API Design | RESTful under `/api/v1/*` |
| ORM | Supabase JS client (raw SQL via RPC where needed) |

## Directory Structure

```
app/api/v1/
├── auth/
│   ├── signup/route.ts
│   ├── callback/route.ts
│   └── me/route.ts
├── workspaces/
├── projects/
├── modules/
├── user-stories/
├── acceptance-criteria/
├── atcs/
├── tests/
├── runs/
├── bugs/
└── access-tokens/
```

## Database

| Attribute | Value |
|---|---|
| Type | PostgreSQL 16 |
| Provider | Supabase (Cloud MVP) |
| Migrations | `supabase/migrations/` directory |
| ORM | None — raw Supabase JS client |
| RLS | Enforced per table with `workspace_id` policy |
| Search | `tsvector` GIN indexes (MVP) |

## Auth Flow

1. User visits protected route → `middleware.ts` intercepts
2. `createServerClient` with `SUPABASE_URL` + `SUPABASE_ANON_KEY` from env
3. `supabase.auth.getUser()` checks session cookie
4. If no user + protected route → redirect to `/login?next={path}`
5. Protected prefixes: `/projects/*`, `/onboarding/*`
6. Public prefixes: `/login`, `/auth`, `/api/auth`

## Dependencies

| Package | Purpose |
|---|---|
| `@supabase/supabase-js` | Database + Auth client |
| `@supabase/ssr` | Server-side rendering auth helpers |
| `@asteasolutions/zod-to-openapi` | OpenAPI generation from Zod |
| `zod` | Validation schemas |

## Run/Test Commands

```bash
# Development
bun run dev              # next dev — local dev server

# Build
bun run build            # next build

# Type check
bun run typecheck        # tsc --noEmit

# Code quality
bun run lint:check       # ESLint
bun run format:check     # Prettier

# Full repo check
bun run repo:check       # format + lint + types + vars + skills

# Environment validation
bun run vars:env:check
```

## Auth Credentials

Auth stored in `.env`:
```
SUPABASE_URL=<project-url>
SUPABASE_ANON_KEY=<anon-key>
SUPABASE_SERVICE_ROLE_KEY=<service-role-key>
```

## Discovery Gaps

- [ ] Live Supabase migration files not audited (check `supabase/migrations/`)
- [ ] RLS policy definitions not verified (need migration file or `[DB_TOOL]`)
- [ ] API rate limiting not implemented yet (planned in NFRs)
