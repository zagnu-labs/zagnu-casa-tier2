# Access Control and Account Management

How the platform avoids shared and default accounts (e.g. "root", "admin",
"sa"), and how access is authenticated and authorized. Covers the application,
data stores, and infrastructure.

Last reviewed: 2026-06.

## 1. Application — no shared accounts

- **Per-user identity.** Every end user authenticates with their own unique
  account; authentication is enforced server-side at the `account-registry`
  boundary. There is no generic, shared, or "demo" login.
- **No built-in admin/superuser account.** The application ships with no default
  "admin" account and no shared administrative login.
- **Server-side enforcement.** Access controls are enforced at trusted
  enforcement points (the API gateway and per-service authorization), never on
  the client. User-facing endpoints require a verified, per-user session token
  (RS256 JWT); internal/administrative endpoints are not exposed on the public
  internet.
- **No default credentials.** The application has no built-in or default
  passwords of any kind.

## 2. Data stores — no default accounts

- The database is **not** accessed through any default vendor account such as
  `sa`, `root`, or the default `postgres` superuser. Access uses a **named**
  application account with credentials that are unique per environment and
  injected from the secret store at deploy time.
- Redis and Kafka are reachable only inside the private VPC; Kafka additionally
  enforces SASL/SCRAM authentication. No default broker/cache accounts are used.

## 3. Infrastructure — no shared/default accounts

- **Cloud (AWS):** the account **root user is MFA-protected and not used** for
  routine operations; day-to-day access is through scoped IAM users/roles.
- **Kubernetes:** workloads run under **per-deployment service accounts** with
  namespaced RBAC; the application runtime does not use a shared cluster-admin.
- **OS / containers:** there are no shared interactive OS logins; all changes go
  through automated CI/CD pipelines rather than shared SSH access.

## 4. Credential hygiene

- Credentials are unique per service/environment, randomly generated, and stored
  in the managed secret store (Kubernetes Secrets / CI encrypted secrets).
- No credential is hard-coded in source or shared across unrelated components.

## Summary

No shared or default accounts (e.g. `root`, `admin`, `sa`) are present.
Application access is per-user with server-side enforcement; data-store and
infrastructure access uses named, least-privilege identities, with cloud root
access protected by MFA and reserved for break-glass use only.
