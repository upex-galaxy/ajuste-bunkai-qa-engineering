# Non-Functional Specifications — Bunkai (QA Context)

> Source: target `.context/SRS/non-functional-specs.md`. MVP targets.

---

## Performance

| Metric | Target (MVP) | Test Method |
|---|---|---|
| LCP on Project View | < 2.0s p75 | Playwright trace with performance API |
| API single-entity read | < 200ms p95 | k6 / Playwright API testing |
| API listing (50 rows) | < 500ms p95 | Pagination + load test |
| Tree query (500 modules / 5000 ATCs) | < 800ms p95 | Seed data + recursive CTE timing |
| Realtime propagation | < 1.5s p95 | WebSocket latency measurement |
| Concurrent users per workspace | 50 (MVP target) | Load test with gradual ramp-up |
| Initial JS bundle | ≤ 300KB gzipped | Lighthouse / WebPageTest |

## Security

| Concern | Approach | Test Method |
|---|---|---|
| AuthN | Supabase Auth (JWT, OAuth, magic link) | Test all auth methods + token expiry |
| AuthZ | RLS policies per `workspace_id` | Cross-workspace access attempts → 403 |
| Input validation | Zod schemas server + client | Fuzz endpoints with invalid payloads |
| XSS | Markdown sanitized via `rehype-sanitize` | Inject `<script>` in ATC description |
| Rate limits | 100 req/min write, 600 req/min read | Burst test → 429 with `Retry-After` |
| CSP | Strict, nonces for inline scripts | Verify CSP headers in Playwright |
| Secrets | Environment vars only | Verify no hardcoded credentials in source |

## SEO (MVP)
- N/A for MVP — no public-facing pages beyond marketing
- API endpoints return `Cache-Control: private` with 60s revalidation

## Accessibility (WCAG 2.1 AA)
- Contrast: all token combinations verified in design tokens
- Keyboard: every interactive element reachable via keyboard (Cmd/Ctrl+K command palette)
- Screen reader: semantic HTML, ARIA labels on complex widgets (Tree, Table)

## Reliability

| Concern | Approach | Test Method |
|---|---|---|
| Error handling | Machine-readable `code` in error responses | Verify every error path returns `{success: false, error: {code, message}}` |
| Idempotency | `idempotency_key` on POST endpoints | Duplicate POST → same resource ID (not 409) |
| Rate limiting | Token-scoped, per-workspace | Write burst → 429, read burst → 429 |
| Data persistence | Supabase managed (AES-256 at rest) | Verify backups exist (out of scope for MVP) |

## Discovery Gaps

- [ ] Actual performance numbers (need staging deployment to measure)
- [ ] Supabase RLS policies implementation status
- [ ] Backup/disaster recovery plan for Cloud edition
- [ ] Observability (logging, metrics) not fully specified
