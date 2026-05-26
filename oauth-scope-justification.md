# OAuth Scope Justification

This document justifies the Google OAuth 2.0 scope requested by the Zagnu application, as required for the CASA Tier 2 assessment of restricted scopes.

---

## Scopes requested

The application requests **exactly one** Google OAuth scope:

| Scope | Sensitivity tier |
|---|---|
| `https://www.googleapis.com/auth/gmail.readonly` | Restricted |

No other Gmail or Google scopes are requested. The application does not use `gmail.modify`, `gmail.compose`, `gmail.send`, `mail.google.com`, `gmail.insert`, or any of the Gmail Add-on scopes.

---

## Why this scope is required

Zagnu's core feature ("attestation") detects and verifies inbound payment-confirmation emails on behalf of the user. To do this, the backend must be able to:

1. **List messages** in the user's inbox filtered by sender domain, subject, and date (`Users.Messages.List` with `from:`, `subject:`, `after:` operators).
2. **Read the full message body** of payment notifications to extract structured payment data — amount, transaction ID, sender name, recipient handle, memo (`Users.Messages.Get` with `Format("full")`).
3. **Read the user's primary email address** to identify the connected account (`Users.GetProfile`).

All three operations are read-only and are covered by the single `gmail.readonly` scope. No other scope grants read access to the message body, which is required for the extraction step.

---

## What the application does NOT do

To make the scope of access explicit, the application **never**:

- Sends, modifies, drafts, or deletes any email.
- Marks messages as read, archives, or moves them.
- Modifies labels, filters, or any Gmail settings.
- Stores the full content of any email beyond the transient request/response cycle of a single extraction call.
- Shares email content with any third party other than OpenAI (for structured field extraction — see `pii-data-flow.md`).
- Accesses messages outside the search filters defined by the active payment strategy (e.g., for the Venmo strategy: `from:venmo.com subject:"paid"`).

---

## Alternatives considered

| Alternative scope | Why it was not chosen |
|---|---|
| `gmail.metadata` | Provides headers only, no body. The application's core extraction step requires the email body to identify transaction data (amount, ID, memo). Without the body, the feature cannot work. |
| Manual forwarding to a Zagnu address | Would require users to set up filter rules to forward payment emails. Higher friction, less reliable, and creates a secondary inbox the user does not control. Not viable for the intended user experience. |
| `https://mail.google.com/` (full access) | Grants read, write, send, and delete — vastly more privileges than required. Violates the principle of least privilege. Rejected. |
| `gmail.modify` | Grants compose and send, which we do not need. Rejected. |

The chosen `gmail.readonly` scope is the minimum scope that enables the required feature.

---

## Token handling

OAuth access and refresh tokens are stored encrypted at rest in the application's PostgreSQL database (Aurora). Tokens are used exclusively server-side and are never transmitted to the user's browser or any third party. Refreshed tokens are persisted back to the database; revoked sessions invalidate the stored tokens and emit a `session.invalidated` event.

When a user disconnects their Google account (either via the Zagnu app or via Google's "Third-party apps with account access" page), the application receives an auth error on the next API call and invalidates the local session.

Full token storage and lifecycle details are documented in `1.1-architecture.md` (Section 6 — Gmail OAuth flow) and `encryption.md`.
