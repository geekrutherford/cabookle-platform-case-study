# Authentication & Security Overview

This document describes the authentication and security posture of the Cabookle platform **at a high
level only**. It deliberately omits anything that would be useful to an attacker, no token formats,
no hashing parameters, no thresholds, no key material, no implementation specifics.

---

## Authentication model

Cabookle uses a **hybrid** approach: short-lived JWT access tokens for API calls, layered on top of a
**server-side session** represented by an opaque HttpOnly cookie.

### Registration & verification

1. A user registers with email and password. The account is created **inactive** and **unverified**.
2. The platform stores a one-time, expiring verification token and emails a verification link.
3. Clicking the link verifies the email, activates the account, and provisions the user's organization
   and product entitlements automatically.
4. Only a verified, active account can sign in.

The same one-time, expiring-token pattern is used for password resets.

### Sign-in & sessions

- On successful sign-in, the platform creates a **server-side session** and sets an **HttpOnly,
  Secure, SameSite** cookie scoped to the shared parent domain, containing only an opaque session id.
- The session has a bounded lifetime and **sliding expiration**: it is only extended on activity, and
  only once more than half its lifetime has elapsed.
- The platform validates the session server-side and returns a **fresh, short-lived JWT** on each
  request. No token ever travels in a URL.
- Sign-out **revokes** the server-side session and clears the cookie immediately.
- Expired and revoked sessions are purged by a background job.

### Credential security

- Passwords are stored only as salted, one-way hashes via a dedicated password hashing service.
- Repeated failed sign-in attempts trigger account lockout.
- Public forms are protected with **reCAPTCHA v3**, verified server-side against a configurable score
  threshold, and auth endpoints are **rate-limited**.

---

## Service-to-service security

Product services call platform endpoints (for example, the email relay) using an **internal API key**
sent in a dedicated header. Each service gets its own key so keys can be rotated independently, and
the relay is **rate-limited per client**. This keeps machine-to-machine calls authenticated without
reusing user credentials or cookies.

---

## Defense in depth

| Layer | Measure |
|---|---|
| Transport | TLS terminated at the hosting edge; `Secure` cookies; HSTS headers. |
| App hardening | Security headers middleware (HSTS, Content Security Policy, and related headers) applied to responses. |
| Bot & abuse | reCAPTCHA v3 on public forms; fixed-window rate limiting on auth and email endpoints. |
| Account | Email verification, one-time expiring tokens, account lockout, forced password reset. |
| Service-to-service | Per-service internal API keys + rate limiting. |
| Secrets | Secrets live outside the codebase (user-secrets in development, environment variables in hosting); automated secret scanning in CI. |
| Supply chain | Static analysis (SAST) and dependency updates run automatically in CI. |

---

## What is intentionally not shared

This case study omits details that would meaningfully help someone attack a Cabookle deployment:

- JWT signing key material, issuer/audience specifics, and token lifetime numbers.
- Password hashing algorithm parameters and iteration counts.
- Exact lockout and rate-limit thresholds.
- reCAPTCHA score thresholds and site keys.
- Internal API key values and header/endpoint specifics beyond what is necessary to explain the model.
- Cookie flags beyond the standard security properties described above.

The point is to show **how** the security model is structured, not to provide a recipe.
