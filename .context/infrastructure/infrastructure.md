# Infrastructure — Bunkai (QA Context)

> Source: target code analysis (`.agents/project.yaml`, `.env.example`, `.github/workflows/`, `Dockerfile`).

---

## CI/CD

| Attribute | Value |
|---|---|
| Provider | Not configured (pre-MVP) |
| Workflows | None found in `.github/workflows/` |
| Local checks | `bun run repo:check` (format + lint + types + vars + skills) |
| Pre-commit | Husky (lint-staged) |

## Deployment

| Environment | URL | Provider |
|---|---|---|
| Local | `http://localhost:3000` | Next.js dev server |
| Staging | `https://staging-upexbunkai.vercel.app` | Vercel (staging branch) |
| Production | `https://upexbunkai.vercel.app` | Vercel (main branch) |

## Environments

| Env | Web URL | API URL | Supabase |
|---|---|---|---|
| local | `http://localhost:3000` | `http://localhost:3000/api` | `fmbpikzpkafptqximhxn` |
| staging | `https://staging-upexbunkai.vercel.app` | `https://staging-upexbunkai.vercel.app/api` | `fmbpikzpkafptqximhxn` |
| production | `https://upexbunkai.vercel.app` | `https://upexbunkai.vercel.app/api` | `fmbpikzpkafptqximhxn` |

## Infrastructure Setup

- **DNS**: bunkai.io (primary), bankai.io (defensive)
- **CDN**: Vercel Edge Network
- **Database**: Supabase (PostgreSQL 16, single-project tenancy for MVP)
- **Storage**: Cloudflare R2 (S3-compatible, egress-free)
- **Auth**: Supabase Auth
- **Monitoring**: Sentry (errors), PostHog (analytics)
- **Secrets**: Vercel Environment Variables (staging + production), `.env` (local)

## Required .env Keys

```
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ATLASSIAN_URL=
ATLASSIAN_EMAIL=
ATLASSIAN_API_TOKEN=
TAVILY_API_KEY=
CLOUDFLARE_R2_ACCESS_KEY=
CLOUDFLARE_R2_SECRET_KEY=
```

## Discovery Gaps

- [ ] No CI/CD pipelines — blocking for regression testing
- [ ] Vercel deployment not verified (no access to Vercel dashboard)
- [ ] Cloudflare R2 credentials not configured
- [ ] Docker Compose self-hosted distribution (Phase 2, not available yet)
- [ ] No backup/disaster recovery documented for Cloud edition
