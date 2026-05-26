# DAST Remediation Report

This report documents the disposition of every finding from the initial Veracode DAST scan performed against `https://www.zagnu.com` as part of the CASA Tier 2 assessment. Of the 7 total findings (5 Info, 2 Low), 4 have been fixed in code and 3 have been formally risk-accepted with justification.

**Scan target**: `https://www.zagnu.com` (production)
**Scan tool**: Veracode DAST Essentials (Google-provided)
**Codebase**: `lp-pwa` (Next.js on Vercel)

---

## Summary table

| # | Finding | CWE | Severity | Status | Evidence |
|---|---|---|---|---|---|
| 1 | Cross-Origin-Opener-Policy Header Missing or Invalid | — | Info | **Fixed** | `next.config.ts` |
| 2 | Cross-Domain Misconfiguration (CORS wildcard) | — | — | **Fixed** | `next.config.ts` |
| 3 | Proxy Disclosure | CWE-204 | Low | **Partially fixed + Risk Accepted** | `next.config.ts` + platform constraint |
| 4 | Cookie-based Session Behavioral Discrepancy | CWE-205 | Info | **Risk Accepted** | Platform cookie (`_vcrcr`) |
| 5 | Modern Web Application | — | Info | **No action required** | Informational notice |
| 6 | Retrieved from Cache | CWE-525 | Info | **Fixed** | `next.config.ts` |
| 7 | Storable but Non-Cacheable Content | CWE-524 | Info | **Risk Accepted** | Static asset caching by design |

---

## Finding 1 — Cross-Origin-Opener-Policy Header Missing

**Severity**: Info
**Status**: Fixed

**Original observation**: The COOP header was missing from responses on `https://www.zagnu.com`. COOP isolates browsing contexts and protects against side-channel attacks via cross-origin window references.

**Fix**: Added the COOP header to the catch-all rule in the Next.js headers configuration.

**File**: `next.config.ts`

```ts
{
  source: "/(.*)",
  headers: [
    // ...existing headers
    {
      key: "Cross-Origin-Opener-Policy",
      value: "same-origin-allow-popups",
    },
  ],
}
```

`same-origin-allow-popups` is used (instead of strict `same-origin`) because the OAuth consent flow may open popups under some browser configurations.

**Verification**: `curl -sI https://www.zagnu.com/` returns `Cross-Origin-Opener-Policy: same-origin-allow-popups`.

---

## Finding 2 — Cross-Domain Misconfiguration (CORS wildcard)

**Severity**: Info
**Status**: Fixed

**Original observation**: Responses included `Access-Control-Allow-Origin: *`. This wildcard was injected by Vercel's edge by default on prerendered HTML pages and would allow any origin to read response bodies via cross-origin XHR/fetch.

**Fix**: Explicitly override the `Access-Control-Allow-Origin` header on every response with the canonical production origin.

**File**: `next.config.ts`

```ts
{
  source: "/(.*)",
  headers: [
    // ...existing headers
    {
      key: "Access-Control-Allow-Origin",
      value: "https://www.zagnu.com",
    },
  ],
}
```

**Verification**: `curl -sI https://www.zagnu.com/` returns `Access-Control-Allow-Origin: https://www.zagnu.com` (no longer `*`).

---

## Finding 3 — Proxy Disclosure

**Severity**: Low (CWE-204)
**Status**: Partially fixed + Risk Accepted for remainder

**Original observation**: Response headers revealed the hosting platform (`server: Vercel`, `x-vercel-id`, `x-vercel-cache`, `x-nextjs-*`). An attacker can fingerprint the technology stack and look up known CVEs.

**Fix applied**: Disabled the `X-Powered-By: Next.js` header at the Next.js level.

**File**: `next.config.ts`

```ts
const nextConfig: NextConfig = {
  poweredByHeader: false,
  // ...
};
```

**Risk accepted for**: `server: Vercel`, `x-vercel-id`, `x-vercel-cache`, `x-nextjs-*`.

**Justification**: These headers are injected by the Vercel edge platform and cannot be modified or suppressed at the application level. The headers do not expose any vulnerability themselves and are common across all Vercel-hosted applications. Since the platform is fully managed, any vulnerability in the underlying infrastructure would be addressed by Vercel as part of their normal patching process.

---

## Finding 4 — Cookie-based Session Behavioral Discrepancy

**Severity**: Info (CWE-205)
**Status**: Risk Accepted

**Original observation**: The scanner detected that dropping the `_vcrcr` cookie produced a different response than the baseline, indicating that the cookie influences server-side behavior.

**Justification**: The `_vcrcr` cookie is set by the Vercel edge platform for internal routing purposes (canary deployments, edge cache vary, gradual rollouts). It cannot be modified or removed at the application level. The behavioral discrepancy observed is platform-managed routing logic, not application-controlled authentication or authorization. The application does not rely on this cookie for any security decisions.

---

## Finding 5 — Modern Web Application

**Severity**: Info
**Status**: No action required

**Original observation**: The scanner detected that the application is a single-page application (Next.js) and recommended using the AJAX Spider crawler for better URL coverage.

**Justification**: This is an informational notice that pertains to the scanner's own crawling strategy and does not identify any vulnerability in the application. The Remedies section of the report explicitly states: "This is an informational alert and so no changes are required."

---

## Finding 6 — Retrieved from Cache

**Severity**: Info (CWE-525)
**Status**: Fixed

**Original observation**: Pages including `/onboarding` were returned with `Cache-Control: public, max-age=0, must-revalidate` and had an `Age` header set, indicating a shared cache (Vercel edge) had stored them. Shared caching of user-specific pages could leak personal data to another user under unusual proxy configurations.

**Fix**: Added explicit no-store, private cache directives for user-flow routes.

**File**: `next.config.ts`

```ts
{
  source: "/onboarding",
  headers: [
    {
      key: "Cache-Control",
      value: "no-cache, no-store, must-revalidate, private",
    },
  ],
},
{
  source: "/secure/:path*",
  headers: [
    {
      key: "Cache-Control",
      value: "no-cache, no-store, must-revalidate, private",
    },
  ],
},
{
  source: "/api/:path*",
  headers: [
    {
      key: "Cache-Control",
      value: "no-cache, no-store, must-revalidate, private",
    },
  ],
}
```

The other URLs originally flagged in this finding (`/` and `/web.manifest`) are public-only resources without any user data; they are intentionally cacheable for performance. They are addressed by the risk acceptance under Finding 7.

**Verification**: `curl -sI https://www.zagnu.com/onboarding` returns `Cache-Control: no-cache, no-store, must-revalidate, private`.

---

## Finding 7 — Storable but Non-Cacheable Content

**Severity**: Info (CWE-524)
**Status**: Risk Accepted

**Original observation**: The scanner flagged both immutable static assets (`/_next/static/chunks/*`, `/_next/static/css/*`, `/_next/static/media/*` with `max-age=31536000`) and public HTML/text resources (`/`, `/robots.txt`, `/sitemap.xml`, `/web.manifest` with `max-age=0`) as storable in shared caches.

**Justification**: Two categories of URLs are flagged:

1. **`_next/static/*` assets** (JS, CSS, fonts) carry a content-based hash in the filename. The aggressive `max-age=31536000` caching is intentional and correct: any change to the underlying content produces a new filename, so caches are never served stale or wrong content. This is standard Next.js behavior and the security best-practice for hashed static assets.

2. **Public HTML/text resources** (`/`, `/robots.txt`, `/sitemap.xml`, `/web.manifest`) are identical for every visitor and contain no user-specific or sensitive data. The `max-age=0, must-revalidate` setting ensures shared caches always revalidate with the origin before serving. User-flow pages that may render user-specific data (`/onboarding`, `/secure/*`, `/api/*`) are explicitly marked `Cache-Control: no-cache, no-store, must-revalidate, private` (see Finding 6) and excluded from shared caching.

---

## Re-validation

After the fixes above are deployed to production, a re-scan is expected to:
- Confirm Findings 1, 2, 6 are no longer reported.
- Confirm Finding 3 is reduced (no `X-Powered-By` header), with the remainder formally documented as platform-managed.
- Confirm Findings 4, 5, 7 remain in the same status (informational / risk-accepted).

The re-validation summary will be filed as `revalidation-summary.md` once the rescan is executed.
