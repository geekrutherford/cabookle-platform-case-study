# Difficult Engineering Problems

This document highlights a handful of non-trivial problems the platform had to solve, and the approach
taken for each. The emphasis is on the *shape* of the problem and the trade-off, not on
implementation specifics.

---

## 1. Multi-tenant data isolation

**Problem.** Many independent organizations share one application and one database. A single missing
`WHERE organization_id = ...` could leak one school's student and circulation data to another.

**Approach.** Make the tenant the root of the data graph and enforce a single invariant: **every
tenant-scoped query is filtered by the caller's organization id.** Identity is established once, at
the platform, and carried as claims in a short-lived token; products trust and re-apply that tenant
id everywhere. The pattern is encoded in a shared query guard (see [code samples](code-samples.md))
so it isn't left to each developer to remember.

**Trade-off.** Shared-database scoping keeps operations simple (one schema, one backup, instant
signup) but moves the isolation guarantee into application code — which is exactly why the guard is a
first-class abstraction rather than a convention.

---

## 2. Cross-product single sign-on

**Problem.** Three separate web apps should feel like one product: sign in once, move freely between
them, and never pass a token in a URL (which leaks into logs and browser history).

**Approach.** A hybrid token model:
- An **HttpOnly, Secure, SameSite cookie** on the shared parent domain carries only an opaque session id.
- A **server-side session** holds the authoritative state, so sessions can be revoked immediately.
- A **sliding lifetime** extends the session only on activity and only past the halfway point, so
  active users aren't logged out but idle sessions expire.
- On each call, the platform validates the session and issues a **fresh, short-lived JWT** the products
  use for API access.

**Trade-off.** Server-side sessions mean a database lookup per session validation — accepted in
exchange for instant revocation and centralized lifetime control.

---

## 3. Reliable transactional email

**Problem.** Sending email synchronously in the request path couples two systems with very different
failure modes: if the email provider is slow or down, signup and checkout shouldn't fail or hang.

**Approach.** A **transactional outbox**:
- The request writes a queued email row in the same transaction as the business operation, then returns.
- A **background worker** polls for unsent messages and delivers them with **exponential backoff**
  retries (bounded attempt count).
- Delivery is idempotent enough that a crash between "deliver" and "mark sent" can't silently lose a
  message or spam duplicates.

**Trade-off.** Email becomes eventually-consistent (a small delay) in exchange for not blocking
requests and not losing messages.

---

## 4. Service-to-service authentication + abuse control

**Problem.** Product services need to trigger platform-side actions (like sending email) without
reusing user credentials, while still being protected from abuse.

**Approach.** Per-service **internal API keys** (so keys rotate independently) validated in dedicated
middleware, combined with **per-client rate limiting** on the relay endpoints. A compromised or noisy
caller is isolated and throttled without affecting others.

---

## 5. Bot protection without hurting real users

**Problem.** Public forms (register, login, forgot-password, contact) attract bots and credential
stuffing, but friction hurts legitimate teachers and librarians.

**Approach.** Invisible **reCAPTCHA v3** scoring on public forms (verified server-side against a
configurable threshold) plus **rate limiting** and **account lockout**. Legitimate users see no
challenge; automated traffic is scored and throttled. A kill switch lets the team disable verification
in an emergency without redeploying.

---

## 6. Secret hygiene in CI

**Problem.** Secrets and vulnerabilities can slip into a repository over time — and a public-facing
profile must be able to prove the house is in order.

**Approach.** Automated, always-on checks in CI: **secret scanning** on every push/PR, **SAST** static
analysis, and **automated dependency updates** with grouped, reviewable PRs. Secrets themselves live
outside the repo entirely.

---

## 7. Zero-downtime schema evolution

**Problem.** Shipping database changes to a live multi-tenant app without manual steps or downtime.

**Approach.** **EF Core migrations checked into source control**, applied automatically at startup. A
deploy is self-contained: bring up the new image and it migrates the schema before serving traffic.

---

## 8. Grounding AI answers to a tenant's own data

**Problem.** Natural-language features must be genuinely useful *and* safe: the model must never
recommend a book the library doesn't own, and must never see another tenant's data.

**Approach.** A retrieval-then-generate pattern (see [AI & natural language](ai-insights.md)) where
retrieval is **scoped to the organization first**, and generation is constrained to the retrieved
context. Deterministic queries take a direct path, keeping the generative step off the hot path.

**Trade-off.** The grounding constraint is more engineering than a naive "just ask the model" approach
— but it's what makes the feature trustworthy.
