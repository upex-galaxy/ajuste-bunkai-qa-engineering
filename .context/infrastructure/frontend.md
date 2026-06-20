# Frontend — Bunkai (QA Context)

> Source: target code analysis (`app/` directory, `package.json`, `components.json`, `tailwind.config.ts`).

---

## Runtime

| Attribute | Value |
|---|---|
| Framework | Next.js 15 (App Router) |
| Language | TypeScript 5.9 |
| Rendering | React Server Components + Client Components where needed |
| Styling | Tailwind CSS 3.4 + shadcn/ui |
| Component Library | Radix UI primitives (dialog, dropdown, tabs, tooltip) |
| Icons | lucide-react |
| Table | TanStack React Table |
| Editor | Monaco Editor (`@monaco-editor/react`) |
| Command Palette | cmdk |
| Toasts | sonner |
| Markdown Rendering | react-markdown + rehype-sanitize + remark-gfm |
| Charts/Graphs | Shiki (syntax highlighting) |
| Class Utilities | clsx, tailwind-merge, class-variance-authority |

## Directory Structure

```
app/
├── layout.tsx            # Root layout
├── page.tsx              # Landing/marketing
├── globals.css           # Global styles + Tailwind
├── (app)/                # Protected app routes
├── (auth)/               # Auth pages layout
├── auth/                 # Login, signup, callback
├── api/v1/               # API route handlers
├── design-tokens/        # Design system page
├── invites/               # Workspace invite accept flow
└── qa/                   # Testability guide page
```

## Route Map

| Route | Access | Purpose |
|---|---|---|
| `/` | Public | Landing page |
| `/login` | Public | Sign in |
| `/auth/callback` | Public | OAuth callback |
| `/(app)/projects` | Protected | Project dashboard |
| `/(app)/projects/{id}` | Protected | Project view + tree |
| `/invites/{token}` | Public | Accept workspace invite |
| `/qa` | Public | Software testability guide |
| `/design-tokens` | Public | Design system reference |

## UI Data Flow

- Server Components handle initial data fetch (RSC)
- Client Components handle interactivity (TanStack Table, Monaco, command palette)
- Supabase Realtime pushes updates to Run screen
- React Query / TanStack Query for client-side caching

## Test ID Strategy

No `data-testid` attributes detected in component structure. Will need to add them during `/adapt-framework`.

## Discovery Gaps

- [ ] No `data-testid` or test-selector attributes found (need to add for Playwright)
- [ ] Component tree not fully mapped (need to inspect individual components)
- [ ] i18n strategy not confirmed (no locale files found — likely English-only for MVP)
