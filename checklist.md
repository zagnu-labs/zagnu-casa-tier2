# CASA Tier 2 Progress Checklist

A live tracker for the CASA Tier 2 assessment of the Zagnu application's Google OAuth integration (`gmail.readonly` scope).

---

## Phase 1 — Pre-assessment

- [x] Production endpoints publicly reachable for scanning (`https://www.zagnu.com`, `https://api.zagnu.com`)
- [x] Security headers baseline in place (`X-Frame-Options`, `Referrer-Policy`, etc.)
- [x] HTTPS enforced platform-wide (TLS termination at Vercel edge + AWS ALB)
- [x] OAuth consent screen completed in Google Cloud Console
- [x] App moved to "In production" state in Google Cloud (or test users list maintained)

## Phase 2 — DAST scan (initial)

- [x] Veracode DAST Essentials scan executed against `https://www.zagnu.com`
- [x] Findings report received (7 findings: 5 Info, 2 Low)
- [x] Findings triaged: 4 fixable, 3 risk-accepted

## Phase 3 — Remediation

- [x] Finding 1 (COOP missing) — fixed in `next.config.ts`
- [x] Finding 2 (CORS wildcard) — fixed in `next.config.ts`
- [x] Finding 3 (Proxy disclosure) — `poweredByHeader: false`; remainder risk-accepted
- [x] Finding 6 (Cache control) — added `no-store, private` for user-flow routes
- [x] Fixes deployed to production via Vercel (auto on push to `main`)
- [x] Header verification via `curl -I` post-deploy

## Phase 4 — Documentation pack

- [x] `1.1-architecture.md` — architecture, trust boundaries, data flows
- [x] `dast-remediation.md` — 7 findings with disposition and evidence
- [x] `oauth-scope-justification.md` — why we need `gmail.readonly`
- [ ] `encryption.md` — at-rest and in-transit encryption per component
- [ ] `pii-data-flow.md` — PII inventory and flow
- [ ] `data-retention.md` — retention windows and deletion procedures
- [ ] `revalidation-summary.md` — post-rescan summary (Phase 6)

## Phase 5 — Self-Assessment Questionnaire (SAQ)

- [ ] Repo `zagnu-casa-tier2` published to GitHub as public
- [ ] SAQ form opened in CASA portal
- [ ] Each question answered with `Yes` / `N/A` and a comment linking the relevant doc in the public repo
- [ ] SAQ submitted to assessor (TAC Security or equivalent)

## Phase 6 — Revalidation

- [ ] DAST scan re-executed against production after fixes
- [ ] Re-scan report shows Findings 1, 2, 6 removed; Finding 3 partially reduced
- [ ] `revalidation-summary.md` filed in repo
- [ ] Final submission to assessor for review

## Phase 7 — Approval

- [ ] CASA Tier 2 certificate issued
- [ ] Google OAuth app moved out of "unverified" state
- [ ] User cap removed (no longer limited to 100 test users)
