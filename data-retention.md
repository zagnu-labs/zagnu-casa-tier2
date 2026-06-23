# Data Retention and Deletion

Retention windows and deletion procedures for data handled by the Gmail OAuth
integration (`zmail-services`).

Last reviewed: 2026-06.

## 1. Retention by data type

| Data | Retention | Mechanism |
|------|-----------|-----------|
| Raw email bodies | **Not retained** (zero) | Processed in memory for extraction; never written to storage |
| Short-link codes | **30 minutes** | Redis TTL (automatic expiry) |
| Google OAuth tokens | Lifetime of the active connection | Retained while the user keeps the inbox connected; removed on disconnection/account deletion |
| Extracted payment/attestation records | Retained as business records while the account is active | Removed on account deletion |
| User email address | Same as the records it belongs to | Removed on account deletion |
| Application logs | Standard platform log retention | No L1/L2 secrets in clear text |

## 2. Revocation

A user can revoke Gmail access at any time from their Google account settings.
On revocation (or on an auth failure indicating revocation), `zmail-services`
marks the corresponding session **invalid** so the tokens are no longer used for
inbox reads.

## 3. Deletion procedure

On an account-deletion or inbox-disconnection request, the user's data is removed:

- Delete the user's row from `oauth_tokens` (removes the encrypted tokens and
  email association).
- Delete the user's attestation intents and attestations.
- Redis short-link codes require no action (they auto-expire within 30 minutes).
- Raw email bodies require no action (never persisted).

Because OAuth tokens are stored encrypted (AES-256-GCM), even prior to deletion
they are not readable from the database without the encryption key.

## 4. Notes / ownership

- Retention windows above for tokens and attestation records are governed by the
  active-account lifecycle; any fixed maximum-retention window (e.g. periodic
  purge of long-invalidated tokens) is a policy decision tracked by the platform
  owners and can be enforced with a scheduled job if required.
- No personal data is retained by third parties: Google is the source; OpenAI
  receives email bodies transiently for extraction and is contractually
  prohibited from using them for training.
