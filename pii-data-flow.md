# PII Inventory and Data Flow

Inventory of personal data the platform processes for the Gmail OAuth
integration, and how it flows through the system.

Last reviewed: 2026-06.

## 1. PII inventory

| PII element | Source | Stored? | Where | Purpose |
|-------------|--------|---------|-------|---------|
| User email address | Google profile (after consent) | Yes | PostgreSQL (`oauth_tokens`, attestation tables) | Identify which inbox a token/attestation belongs to |
| Google OAuth tokens | Google OAuth exchange | Yes (encrypted) | PostgreSQL (`oauth_tokens`) | Read the connected inbox |
| Email body of payment notifications | User's Gmail inbox | **No** (transient) | in memory only | Extract payment fields |
| Sender name | Extracted from payment email | Yes | attestation tables | Payment attestation |
| Payment counterparty handle, amount, fee, transaction id, memo | Extracted from payment email | Yes | attestation tables | Payment attestation |

No other categories of personal data (e.g., government IDs, full message
archives, attachments) are collected or stored by `zmail-services`.

## 2. Data flow — Gmail integration

```
User consents (Google OAuth)
        │  authorization code (TLS)
        ▼
zmail-services callback
        │  exchange code → tokens; fetch email address (TLS to Google)
        ▼
Encrypt tokens (AES-256-GCM) ──► PostgreSQL (oauth_tokens)   [tokens at rest, encrypted]
        │
        ▼ (scheduled / on demand)
Read inbox via Gmail API (TLS, gmail.readonly)
        │  candidate email bodies (in memory)
        ▼
Extraction: body sent to OpenAI (TLS) ──► structured fields
        │  raw body discarded (never persisted)
        ▼
Persist minimal derived fields ──► PostgreSQL (attestation tables)
        │
        ▼
Emit event on Kafka (TLS + SASL/SCRAM) for the rest of the platform
```

## 3. Key properties

- **Minimization:** the raw email body is the largest piece of personal data
  touched, and it is **never stored** — only discrete extracted fields are kept.
- **Scope limitation:** Gmail access is read-only (`gmail.readonly`); the
  platform never sends, modifies, or deletes mail.
- **Third parties:** personal data is shared only with Google (source) and
  OpenAI (extraction of payment-email bodies). OpenAI's API terms prohibit using
  this data for model training. No other third parties receive personal data.
- **Boundaries:** tokens never leave the backend; internal services are not
  publicly reachable (see `1.1-architecture.md`).

## 4. User rights

- A user can revoke Gmail access at any time from their Google account settings.
  On revocation, `zmail-services` invalidates the corresponding session.
- Account deletion / disconnection removes the stored tokens and associated
  records (see `data-retention.md`).
