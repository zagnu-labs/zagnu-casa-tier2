# Data Classification

All sensitive data handled by the Zagnu platform (focus: the Gmail OAuth
integration in `zmail-services`), classified into protection levels with the
controls applied to each.

Last reviewed: 2026-06.

## Protection levels

| Level | Definition | Baseline controls |
|-------|------------|-------------------|
| **L1 — Restricted** | Credentials/secrets that grant access to user data or systems. | App-layer AES-256-GCM and/or KMS at rest, TLS 1.2+ in transit, never logged, secret-store only, service-only access. |
| **L2 — Confidential** | Personal data (PII) and financial/payment data. | TLS 1.2+ in transit, storage encryption at rest, access-controlled endpoints, data minimization, not logged in clear. |
| **L3 — Internal** | Operational data with no direct PII value. | TLS in transit, standard access controls. |
| **L4 — Public** | Non-sensitive. | None required. |

## Data inventory

| Data element | Where | Level | At rest | In transit |
|--------------|-------|-------|---------|------------|
| Google OAuth access/refresh tokens | `oauth_tokens` (PostgreSQL) | **L1** | AES-256-GCM (app) + Aurora KMS | TLS to Google + TLS to DB |
| Microsoft OAuth tokens | `oauth_tokens` (PostgreSQL) | **L1** | AES-256-GCM (app) + Aurora KMS | TLS + TLS to DB |
| Service secrets (DB password, OpenAI key, token-encryption key, OAuth client secrets, Kafka creds, Slack webhook) | Kubernetes / CI secrets | **L1** | secret-store encryption | TLS |
| Session credential (bearer) | request header / memory | **L1** | not persisted | TLS |
| User email address | `oauth_tokens`, attestation tables | **L2** | Aurora KMS | TLS |
| Short-link codes | Redis (30-min TTL) | **L3** | ephemeral, opaque random | in-VPC |
| Intent/attestation IDs, status, timestamps | attestation tables | **L3** | Aurora KMS | TLS |

## Notes

- **L1 OAuth tokens** receive defense-in-depth: application-layer AES-256-GCM
  *and* database storage encryption (see `encryption.md`).
- **Data minimization:** raw mailbox content is never stored; only the minimal
  derived attestation fields are persisted (see `pii-data-flow.md`).
- **No secrets in logs:** L1/L2 values are never written to logs in clear text.
- Retention and deletion of each element are covered in `data-retention.md`.
