# Encryption — At Rest and In Transit

How the Zagnu platform protects sensitive data with encryption, per component.
Focus is the Google OAuth / Gmail integration handled by `zmail-services`.

Last reviewed: 2026-06.

## 1. Encryption in transit

All network communication that carries sensitive data uses TLS 1.2+.

| Path | Transport |
|------|-----------|
| End user → Frontend (Vercel) | HTTPS (TLS terminated at Vercel edge) |
| Frontend → Public API gateway | HTTPS (TLS terminated at AWS ALB, ACM-managed certificate) |
| Backend → Google OAuth / Gmail API | HTTPS (TLS 1.2+) |
| Backend → OpenAI API | HTTPS (TLS 1.2+) |
| Backend → PostgreSQL (Aurora) | TLS |
| Backend → Redis (ElastiCache) | In-cluster, private VPC |
| Backend → Kafka (MSK) | TLS + SASL/SCRAM-SHA-512 |

Internal services are not exposed to the public internet; only the gateway is
reachable, and only over HTTPS.

## 2. Encryption at rest

### OAuth tokens — application-layer encryption (AES-256-GCM)

Google (and Microsoft) OAuth `access_token` and `refresh_token` values are
encrypted **at the application layer** before being written to the database,
using **AES-256-GCM** (authenticated encryption, AEAD):

- 256-bit key, supplied via a managed secret (`TOKEN_ENCRYPTION_KEY`), never in code.
- A fresh cryptographically-random 12-byte nonce per value.
- Stored format: `enc:v1:base64(nonce ‖ ciphertext)` — the version prefix supports
  future key/algorithm rotation.
- Implemented in `zmail-services`: `internal/crypto` (cipher) and
  `internal/tknstore` (applied at the persistence boundary — encrypt on write,
  decrypt only in memory on read).

This means the tokens are unreadable directly from the database; recovering them
requires the key, which lives only in the secret store.

### Storage-layer encryption

- **PostgreSQL (Aurora):** storage-level encryption at rest (AWS KMS).
- **Redis (ElastiCache):** holds only opaque short-link codes (30-min TTL), no PII.
- **Kafka (MSK):** broker storage encryption at rest.
- **Secrets:** Kubernetes Secrets and GitHub Actions encrypted secrets; injected
  into pods at deploy time, never committed to source control.

### Defense in depth

OAuth tokens are therefore protected by **two independent layers**:
application-layer AES-256-GCM *and* Aurora storage encryption. A database
compromise alone does not expose usable tokens.

## 3. Key management

- The token-encryption key is a 256-bit random key (`openssl rand -base64 32`),
  stored as an encrypted CI/CD secret and injected as an environment variable.
- The key is never logged, never returned by any API, and never committed.
- Rotation: the `enc:v1:` version prefix allows introducing a new key/version
  while still reading existing values.

## 4. What is NOT stored

Raw email bodies are processed in memory for field extraction and are **never
persisted**. Only the minimal derived fields are stored (see
`data-classification.md` and `pii-data-flow.md`).
