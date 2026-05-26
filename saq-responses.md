# SAQ Responses

Draft answers for the 54-question CASA Tier 2 Self-Assessment Questionnaire (TAC Security portal) for the Zagnu application.

> **Note**: The portal exposes only `Yes / No` toggles, with the comment field used to clarify when a control is "not applicable" (vacuously satisfied). All such cases are marked **Yes** with `(N/A)` called out explicitly in the comment. This is the standard interpretation when an N/A toggle is not available.
>
> Comments reference the public evidence repo at: `https://github.com/zagnu-labs/zagnu-casa-tier2/` (replace with the actual URL once the repo is published).

---

### Q1. Verify documentation and justification of all the application's trust boundaries, components, and significant data flows.

**Applicable**: Yes
**Comment**:
> Architecture, trust boundaries, components, and significant data flows (including the end-to-end Gmail OAuth flow) are documented in https://github.com/zagnu-labs/zagnu-casa-tier2/blob/main/1.1-architecture.md.

---

### Q2. Verify the application does not use unsupported, insecure, or deprecated client-side technologies such as NSAPI plugins, Flash, Shockwave, ActiveX, Silverlight, NACL, or client-side Java applets.

**Applicable**: Yes
**Comment**:
> The application is a modern Next.js / React Progressive Web App. No Flash, Shockwave, ActiveX, Silverlight, NACL, NSAPI plugins, or Java applets are used anywhere in the codebase.

---

### Q3. Verify that trusted enforcement points, such as access control gateways, servers, and serverless functions, enforce access controls. Never enforce access controls on the client.

**Applicable**: Yes
**Comment**:
> All access control decisions are enforced in Go backend microservices (zmail-services, account-registry, payment-engine). The Vercel-hosted frontend is presentational and never serves as a trust boundary for authorization.

---

### Q4. Verify that all sensitive data is identified and classified into protection levels.

**Applicable**: Yes
**Comment**:
> Sensitive data classes (OAuth access/refresh tokens, user email addresses, phone numbers, and extracted payment-transaction data) are identified and described in https://github.com/zagnu-labs/zagnu-casa-tier2/blob/main/1.1-architecture.md (Section 4 — Sensitive data handling).

---

### Q5. Verify that all protection levels have an associated set of protection requirements, such as encryption requirements, integrity requirements, retention, privacy and other confidentiality requirements, and that these are applied in the architecture.

**Applicable**: Yes
**Comment**:
> Encryption (at-rest via AWS-managed KMS on Aurora, ElastiCache, and S3; in-transit via TLS and SASL/SCRAM), retention, and privacy requirements per data class are described in https://github.com/zagnu-labs/zagnu-casa-tier2/blob/main/1.1-architecture.md (Section 4 — Sensitive data handling, and Section 7 — Authentication and authorization).

---

### Q6. Verify that the application employs integrity protections, such as code signing or subresource integrity. The application must not load or execute code from untrusted sources, such as loading includes, modules, plugins, code, or libraries from untrusted sources or the Internet.

**Applicable**: Yes
**Comment**:
> All application code is bundled by Next.js with content-hashed filenames served from the same origin under /_next/static/. The application does not load JavaScript from third-party CDNs at runtime. Deployment artifacts are produced by CI on signed Git commits in protected branches.

---

### Q7. Verify that the application has protection from subdomain takeovers if the application relies upon DNS entries or DNS subdomains, such as expired domain names, out of date DNS pointers or CNAMEs, expired projects at public source code repos, or transient cloud APIs, serverless functions, or storage buckets (*autogen-bucket-id*.cloud.example.com) or similar. Protections can include ensuring that DNS names used by applications are regularly checked for expiry or change.

**Applicable**: Yes
**Comment**:
> All subdomains (zagnu.com, www.zagnu.com, api.zagnu.com) are explicitly provisioned via AWS Route 53 and Vercel domain configuration. No wildcard DNS or dangling CNAMEs to deprovisioned resources. Cloud resource deletions are coordinated with DNS updates to prevent orphaned pointers.

---

### Q8. Verify that the application has anti-automation controls to protect against excessive calls such as mass data exfiltration, business logic requests, file uploads or denial of service attacks.

**Applicable**: Yes
**Comment**:
> Anti-automation is enforced at multiple layers. Edge: the Vercel platform provides DDoS protection and edge rate limiting on the frontend; AWS Shield Standard provides L3/L4 DDoS protection at the ALB fronting the backend. Application layer: per-route rate limits are enforced on the Go backend (Gin middleware) for sensitive endpoints — most notably phone OTP issuance and verification to prevent enumeration and SMS-flood, and the OAuth callback to prevent brute force on short-link codes. Business-logic bound: the requested OAuth scope `gmail.readonly` is scoped to the connected user's own inbox, so even an authenticated automated client cannot mass-exfiltrate data across accounts.

---

### Q9. Verify that files obtained from untrusted sources are stored outside the web root, with limited permissions.

**Applicable**: Yes
**Comment**:
> The application does not accept user file uploads. There are no untrusted files to store, so no exposure to the risk this control addresses.

---

### Q10. Verify that files obtained from untrusted sources are scanned by antivirus scanners to prevent upload and serving of known malicious content.

**Applicable**: Yes
**Comment**:
> The application does not accept user file uploads. There are no untrusted files to scan or serve, so no exposure to the risk this control addresses.

---

### Q11. Verify API URLs do not expose sensitive information, such as the API key, session tokens etc.

**Applicable**: Yes
**Comment**:
> API URLs use only opaque identifiers (UUIDs for intents and attestations, random base62 codes for short links). Authentication tokens and API keys are transmitted in headers, never in URL paths or query parameters.

---

### Q12. Verify that authorization decisions are made at both the URI, enforced by programmatic or declarative security at the controller or router, and at the resource level, enforced by model-based permissions.

**Applicable**: Yes
**Comment**:
> URI-level authorization: the API gateway routes traffic by URL prefix and each prefix maps to an authorization tier — `/public/*` is unauthenticated, `/private/*` requires a valid user session, `/internal/*` is restricted to in-cluster callers. Authentication middleware rejects requests that do not match the tier requirement before they reach a handler. Resource-level authorization: handlers additionally verify that the authenticated user owns (or has explicit permission on) the requested resource by comparing the session-derived identity against the resource's owner record before any read or write.

---

### Q13. Verify that enabled RESTful HTTP methods are a valid choice for the user or action, such as preventing normal users using DELETE or PUT on protected API or resources.

**Applicable**: Yes
**Comment**:
> Only the HTTP methods required by each route are registered (no implicit OPTIONS/HEAD beyond CORS preflight). State-changing endpoints (POST, DELETE) require an authenticated session; DELETE /attestations/intents/:id is restricted to the intent's owner.

---

### Q14. Verify that the application build and deployment processes are performed in a secure and repeatable way, such as CI / CD automation, automated configuration management, and automated deployment scripts.

**Applicable**: Yes
**Comment**:
> Deployments are performed by GitHub Actions on push to main. Container images are built deterministically and stored in AWS ECR. Helm charts in the infra repo declare the full Kubernetes deployment state. The main branch is protected with required pull request reviews and status checks.

---

### Q15. Verify that the application, configuration, and all dependencies can be re-deployed using automated deployment scripts, built from a documented and tested runbook in a reasonable time, or restored from backups in a timely fashion.

**Applicable**: Yes
**Comment**:
> Infrastructure is declared as code: Terraform for AWS resources, Helm for Kubernetes deployments. The full stack can be recreated in a new environment from these scripts. Aurora and ElastiCache use AWS's automated backup features for point-in-time recovery.

---

### Q16. Verify that authorized administrators can verify the integrity of all security-relevant configurations to detect tampering.

**Applicable**: Yes
**Comment**:
> All deployment configuration (Helm values, Terraform state) is versioned in Git in the private infra repo. Helm release history records the installed values for each deployment; drift between Git and the live cluster can be detected by running `helm diff`. Vercel keeps an immutable deployment log per push.

---

### Q17. Verify that web or application server and application framework debug modes are disabled in production to eliminate debug features, developer consoles, and unintended security disclosures.

**Applicable**: Yes
**Comment**:
> Go services compile to a release binary with no debug endpoints (no pprof, no admin console) registered. Next.js builds run in production mode on Vercel — no React DevTools messages, no Webpack debug output. The Gin router runs in release mode.

---

### Q18. Verify that the supplied Origin header is not used for authentication or access control decisions, as the Origin header can easily be changed by an attacker.

**Applicable**: Yes
**Comment**:
> The Origin header is consumed only by the CORS middleware for preflight validation. It is never used as an authentication or authorization signal.

---

### Q19. Verify that user set passwords are at least 12 characters in length

**Applicable**: Yes
**Comment**:
> The application does not implement user-set passwords. Authentication is via Google OAuth 2.0 (for Gmail integration) and phone number + SMS OTP (for the user account). There is no password length policy to violate because passwords do not exist in the system.

---

### Q20. Verify system generated initial passwords or activation codes SHOULD be securely randomly generated, SHOULD be at least 6 characters long, and MAY contain letters and numbers, and expire after a short period of time. These initial secrets must not be permitted to become the long term password.

**Applicable**: No
**Comment**:
> Partially compliant. SMS OTPs for phone verification are randomly generated by Telnyx (the SMS provider, which uses a cryptographically secure RNG), are valid for 5 minutes, are single-use, and cannot be repurposed as a long-term credential. However, the OTPs are currently 5 digits long rather than the 6-character minimum specified by the control. Compensating controls in place: short 5-minute expiry, single-use enforcement, rate limiting on OTP verification attempts, and lockout after repeated failures.

---

### Q21. Verify that passwords are stored in a form that is resistant to offline attacks. Passwords SHALL be salted and hashed using an approved one-way key derivation or password hashing function. Key derivation and password hashing functions take a password, a salt, and a cost factor as inputs when generating a password hash. ([C6](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> The application does not store user passwords (authentication is via Google OAuth 2.0 and phone-based SMS OTP). There is no password material subject to offline attack, so the control is fully met.

---

### Q22. Verify shared or default accounts are not present (e.g. "root", "admin", or "sa").

**Applicable**: Yes
**Comment**:
> No shared or default accounts are present in any environment. Database users are created per-service with unique credentials. Kubernetes service accounts are scoped per-deployment. Cloud provider root accounts are MFA-protected and not used for routine operations.

---

### Q23. Verify that lookup secrets can be used only once.

**Applicable**: Yes
**Comment**:
> Phone OTPs are single-use and invalidated after successful verification or expiry. Short-link codes stored in Redis are semantically single-use (consumed on first resolution by the OAuth redirect flow) and additionally expire by TTL (30 minutes).

---

### Q24. Verify that the out of band verifier expires out of band authentication requests, codes, or tokens after 10 minutes.

**Applicable**: Yes
**Comment**:
> Phone OTP codes (delivered via Telnyx SMS) expire 5 minutes after issuance, well under the 10-minute requirement. OAuth access tokens follow Google's standard 1-hour lifetime and are refreshed by the library before use.

---

### Q25. Verify that the initial authentication code is generated by a secure random number generator, containing at least 20 bits of entropy (typically a six digital random number is sufficient).

**Applicable**: No
**Comment**:
> Partially compliant. Phone OTPs are currently 5-digit decimal codes generated by Telnyx's CSPRNG, providing approximately 16.6 bits of entropy (log2(10^5)), which is below the 20-bit threshold. Compensating controls in place: 5-minute expiry, single-use enforcement, rate-limited verification attempts (lockout after repeated failures), and rate-limited issuance to prevent SMS-flood enumeration. Short-link codes used in the OAuth redirect flow are 6-character base62 (~36 bits of entropy), generated via Go's crypto/rand and well above the threshold.

---

### Q26. Verify that logout and expiration invalidate the session token, such that the back button or a downstream relying party does not resume an authenticated session, including across relying parties. ([C6](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> Authentication uses stateless JWT access tokens. On logout the frontend deletes the token from client storage, after which the browser back button has no token to resend and cannot resume the authenticated session. On expiration the backend rejects requests carrying a token whose `exp` claim has passed. For the Gmail OAuth integration specifically, disconnecting the account additionally deletes the stored OAuth tokens server-side and emits a `session.invalidated` event consumed by other services.

---

### Q27. Verify that the application gives the option to terminate all other active sessions after a successful password change (including change via password reset/recovery), and that this is effective across the application, federated login (if present), and any relying parties.

**Applicable**: Yes
**Comment**:
> The application does not implement user-set passwords, so there is no "password change" event to handle. The equivalent flow exists for OAuth: when a user disconnects their Google account (either via the Zagnu app or via Google's "Apps with account access" page), the corresponding session is invalidated and removed from the token store, and a session.invalidated event is emitted across all consuming services.

---

### Q28. Verify the application uses session tokens rather than static API secrets and keys, except with legacy implementations.

**Applicable**: Yes
**Comment**:
> End-user authentication uses session tokens issued at login by account-registry. Service-to-service authentication inside the cluster uses scoped internal credentials. Static API keys/secrets exist only for outbound calls to fixed third-party services (Google API, OpenAI) and are never exposed to clients.

---

### Q29. Verify the application ensures a full, valid login session or requires re-authentication or secondary verification before allowing any sensitive transactions or account modifications.

**Applicable**: No
**Comment**:
> Partially compliant. All sensitive operations require a valid, non-expired session token bound to the authenticated user; sessions are invalidated on logout and on OAuth revocation. However, the application does not currently require explicit re-authentication (such as re-issuing a phone OTP) before sensitive transactions like account deletion or disconnecting an OAuth integration. Compensating controls in place: short session lifetimes, server-side validation of every sensitive request, and audit logging of sensitive actions. Remediation plan: introduce a re-OTP step before the account-deletion and OAuth-disconnect endpoints.

---

### Q30. Verify that the application enforces access control rules on a trusted service layer, especially if client-side access control is present and could be bypassed.

**Applicable**: Yes
**Comment**:
> All access control rules are enforced in the Go backend service layer. The Vercel frontend renders UI conditionally based on server-provided data, but UI-level hiding of features is never relied upon for security — the backend independently re-validates every request.

---

### Q31. Verify that all user and data attributes and policy information used by access controls cannot be manipulated by end users unless specifically authorized.

**Applicable**: Yes
**Comment**:
> User identity and role attributes are derived from the server-issued session token and are never accepted from request bodies or query parameters. Foreign-key references supplied by the client (e.g., intent_id, attestation_id) are validated server-side against the session's user before any operation.

---

### Q32. Verify that the principle of least privilege exists - users should only be able to access functions, data files, URLs, controllers, services, and other resources, for which they possess specific authorization. This implies protection against spoofing and elevation of privilege. ([C7](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> Each microservice has its own database role with the minimum permissions it needs (no shared superuser). Kubernetes service accounts are scoped per-deployment with namespaced RBAC. The OAuth scope requested from Google (`gmail.readonly`) is the minimum that enables our use case — read-only, scoped to the connected user's inbox.

---

### Q33. Verify that access controls fail securely including when an exception occurs. ([C10](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> All access control checks default to deny on any exception or unexpected state. Errors return 401/403 status codes via the centralized httputil.Error helper; there are no silent bypasses on panic or unhandled error paths.

---

### Q34. Verify that sensitive data and APIs are protected against Insecure Direct Object Reference (IDOR) attacks targeting creation, reading, updating and deletion of records, such as creating or updating someone else's record, viewing everyone's records, or deleting all records.

**Applicable**: Yes
**Comment**:
> All read/write operations on user-scoped resources include an authorization check that derives the user identity from the session and verifies the requested resource belongs to that user. Bulk endpoints (List) are scoped to the authenticated user's data only. Client-supplied IDs are never trusted without server-side ownership verification.

---

### Q35. Verify administrative interfaces use appropriate multi-factor authentication to prevent unauthorized use.

**Applicable**: Yes
**Comment**:
> Administrative platforms (AWS Console, Vercel Dashboard, GitHub, OpenAI Console, TAC Security portal) all require multi-factor authentication for every team member. The application does not expose any in-app admin interface publicly.

---

### Q36. Verify that the application has defenses against HTTP parameter pollution attacks, particularly if the application framework makes no distinction about the source of request parameters (GET, POST, cookies, headers, or environment variables).

**Applicable**: Yes
**Comment**:
> Gin parses request parameters from explicit, named sources via c.Query, c.Param, c.GetHeader, and c.ShouldBindJSON. The application does not rely on automatic parameter merging across sources, eliminating the ambiguity that enables HPP.

---

### Q37. Verify that the application sanitizes user input before passing to mail systems to protect against SMTP or IMAP injection.

**Applicable**: Yes
**Comment**:
> The application does not send email and does not use SMTP or IMAP for any operation. Gmail integration is strictly read-only via Google's REST API. There is no mail system into which user input could be injected.

---

### Q38. Verify that the application avoids the use of eval() or other dynamic code execution features. Where there is no alternative, any user input being included must be sanitized or sandboxed before being executed.

**Applicable**: Yes
**Comment**:
> The application does not use eval, the Function constructor, or any dynamic code execution feature. Structured outputs from the LLM (OpenAI) are parsed exclusively via encoding/json.Unmarshal against a strict JSON schema with `strict: true`.

---

### Q39. Verify that the application protects against SSRF attacks, by validating or sanitizing untrusted data or HTTP file metadata, such as filenames and URL input fields, and uses allow lists of protocols, domains, paths and ports.

**Applicable**: Yes
**Comment**:
> Outbound HTTP calls are limited to fixed hostnames hard-coded in the application: Google API (gmail.googleapis.com, oauth2.googleapis.com), OpenAI API (api.openai.com), and AWS internal services. No outbound URL is derived from user input. The OAuth redirect URL is also a fixed, environment-pinned value.

---

### Q40. Verify that the application sanitizes, disables, or sandboxes user-supplied Scalable Vector Graphics (SVG) scriptable content, especially as they relate to XSS resulting from inline scripts, and foreignObject.

**Applicable**: Yes
**Comment**:
> The application does not accept, render, or process user-supplied SVG content. There is no user SVG input path through which scriptable content could be introduced.

---

### Q41. Verify that output encoding is relevant for the interpreter and context required. For example, use encoders specifically for HTML values, HTML attributes, JavaScript, URL parameters, HTTP headers, SMTP, and others as the context requires, especially from untrusted inputs (e.g. names with Unicode or apostrophes, such as ねこ or O'Hara). ([C4](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> React (Next.js) handles HTML escaping by default for all values rendered into the DOM. URL parameters are constructed via the URL/URLSearchParams APIs (never string concatenation). The backend exposes JSON-only APIs; no server-side HTML rendering with user content occurs.

---

### Q42. Verify that the application protects against JSON injection attacks, JSON eval attacks, and JavaScript expression evaluation. ([C4](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> JSON output is produced exclusively via encoding/json in Go and JSON.stringify in JavaScript — never via string concatenation. JSON input from external sources (LLM responses, client requests) is validated against strict schemas. The frontend does not use eval or JSON.parse on untrusted strings without validation.

---

### Q43. Verify that the application protects against LDAP injection vulnerabilities, or that specific security controls to prevent LDAP injection have been implemented. ([C4](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> The application does not use LDAP for any authentication or directory operation. There is no LDAP query path through which injection could occur.

---

### Q44. Verify that regulated private data is stored encrypted while at rest, such as Personally Identifiable Information (PII), sensitive personal information, or data assessed likely to be subject to EU's GDPR.

**Applicable**: Yes
**Comment**:
> All databases (AWS Aurora PostgreSQL) and caches (AWS ElastiCache Redis) have encryption-at-rest enabled with keys managed by AWS KMS. Object storage in S3 is also encrypted at rest by default. The architecture overview at https://github.com/zagnu-labs/zagnu-casa-tier2/blob/main/1.1-architecture.md (Section 4) lists which data classes are stored where, and confirms TLS in transit on every hop.

---

### Q45. Verify that all cryptographic operations are constant-time, with no 'short-circuit' operations in comparisons, calculations, or returns, to avoid leaking information.

**Applicable**: Yes
**Comment**:
> All cryptographic operations rely on standard library implementations: Go crypto/subtle for constant-time comparisons, golang.org/x/oauth2 for OAuth token handling, Node crypto for any browser-side needs. The application implements no custom cryptographic primitives or comparisons.

---

### Q46. Verify that random GUIDs are created using the GUID v4 algorithm, and a Cryptographically-secure Pseudo-random Number Generator (CSPRNG). GUIDs created using other pseudo-random number generators may be predictable.

**Applicable**: Yes
**Comment**:
> UUIDs are generated via the github.com/google/uuid Go library (uuid.NewString), which uses crypto/rand internally and produces UUID v4 values. Short-link codes are generated via crypto/rand (base62). No use of math/rand for security-sensitive values.

---

### Q47. Verify that key material is not exposed to the application but instead uses an isolated security module like a vault for cryptographic operations. ([C8](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> Application secrets (database passwords, API keys, OAuth client secret) are stored as GitHub Actions encrypted secrets, injected into Kubernetes pods via Secret resources at deploy time. At-rest encryption keys for Aurora, ElastiCache, and S3 are managed by AWS KMS and never exposed to the application. Key material is not present in source code, container images, or version control.

---

### Q48. Verify that the application does not log credentials or payment details. Session tokens should only be stored in logs in an irreversible, hashed form. ([C9, C10](https://owasp.org/www-project-proactive-controls/#div-numbering))

**Applicable**: Yes
**Comment**:
> Structured logging (Go slog) is configured to record identifiers (intent_id, email, msg_id, attestation_id) but never auth tokens, OTPs, refresh tokens, session cookies, passwords, or card/bank details. The same policy applies to frontend client-side logging.

---

### Q49. Verify the application protects sensitive data from being cached in server components such as load balancers and application caches.

**Applicable**: Yes
**Comment**:
> User-flow routes (/onboarding, /secure/*, /api/*) are served with Cache-Control: no-cache, no-store, must-revalidate, private to prevent shared cache storage. Configuration in `lp-pwa/next.config.ts`; details in https://github.com/zagnu-labs/zagnu-casa-tier2/blob/main/dast-remediation.md#finding-6--retrieved-from-cache.

---

### Q50. Verify that data stored in browser storage (such as localStorage, sessionStorage, IndexedDB, or cookies) does not contain sensitive data.

**Applicable**: Yes
**Comment**:
> Browser storage (localStorage, sessionStorage, IndexedDB, cookies) holds only non-sensitive client state (UI preferences, anonymous session IDs, PWA service-worker registration data). OAuth tokens, refresh tokens, OTPs, and PII never reach the browser.

---

### Q51. Verify that sensitive data is sent to the server in the HTTP message body or headers, and that query string parameters from any HTTP verb do not contain sensitive data.

**Applicable**: Yes
**Comment**:
> Sensitive payloads (OAuth callbacks, intent creation requests, authentication tokens) are transmitted in the HTTP body or in headers. Query string parameters carry only opaque identifiers (UUIDs, random short-link codes) with no PII or secret content.

---

### Q52. Verify accessing sensitive data is audited (without logging the sensitive data itself), if the data is collected under relevant data protection directives or where logging of access is required.

**Applicable**: Yes
**Comment**:
> Access to sensitive operations is logged with session-derived user identity, action, and resource ID. The data itself (OAuth tokens, email bodies) is never logged. Logs are retained for at least 90 days for audit purposes.

---

### Q53. Verify that connections to and from the server use trusted TLS certificates. Where locally generated or self-signed certificates are used, the server must be configured to only trust specific local CAs and specific self-signed certificates. All others should be rejected.

**Applicable**: Yes
**Comment**:
> All public endpoints use TLS certificates issued by trusted authorities: AWS Certificate Manager certificates terminated at the ALB for api.zagnu.com, Vercel-managed certificates for zagnu.com and www.zagnu.com. No self-signed or untrusted CAs are used in production.

---

### Q54. Verify that proper certification revocation, such as Online Certificate Status Protocol (OCSP) Stapling, is enabled and configured.

**Applicable**: Yes
**Comment**:
> OCSP stapling is enabled and managed by the underlying platforms — AWS ACM/ALB and Vercel edge — which automatically perform OCSP queries and staple revocation responses to the TLS handshake on behalf of the application.
