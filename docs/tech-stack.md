# Technology Stack

This document lists the stack and — more importantly — **why** each choice was made. A stack is only
interesting when the reasoning is visible, so each entry below pairs a technology with the decision
that motivated it.

---

## Backend

| Technology | Why |
|---|---|
| **.NET 8** | A mature, high-performance, cross-platform runtime with first-class tooling, ahead-of-time compilation options, and a strong ecosystem for building reliable web APIs. One language for API, worker, and shared libraries. |
| **ASP.NET Core** | The natural choice for a .NET API: robust middleware pipeline, built-in dependency injection, authentication/authorization abstractions, and simple containerization. |
| **EF Core (Pomelo MySQL provider)** | Object-relational mapping that stays in sync with the schema through migrations, so schema changes ship as code and run automatically at startup. Pomelo provides a well-maintained MySQL provider. |

## Data

| Technology | Why |
|---|---|
| **MySQL** | A proven, low-operational-overhead relational database that is easy to run locally and widely supported by managed platforms. The data model (users, organizations, memberships, entitlements, sessions, outbox) is inherently relational. |

## Email

| Technology | Why |
|---|---|
| **Resend** | A developer-friendly transactional email API. Keeping delivery as a single, swappable integration means the platform owns the *policy* (outbox, retries, templates) while an external provider owns deliverability. |

## Frontend

| Technology | Why |
|---|---|
| **Svelte 5** | A small, fast framework whose compiler produces minimal JavaScript. For a focused account/portal SPA, Svelte keeps the codebase lean and the bundle small. |
| **Vite** | Fast dev server and build tooling; standard companion for Svelte. |
| **Tailwind CSS** | Utility-first styling for rapid, consistent UI with minimal custom CSS. |

## Auth & security

| Technology | Why |
|---|---|
| **JWT (shared secret)** | Signed, short-lived tokens issued after session validation. A shared secret lets every product service validate the same tokens without a round-trip to an identity provider. |
| **Server-side sessions + HttpOnly cookie** | The cookie carries only an opaque session id; the authoritative state lives server-side, so sessions can be revoked immediately and lifetime managed centrally. |
| **Google reCAPTCHA v3** | Frictionless bot protection for public forms (register, login, forgot-password, contact), scored server-side with a configurable threshold. |
| **ASP.NET Core rate limiting** | Fixed-window limiters on auth and email endpoints to blunt credential stuffing and abuse. |

## Delivery & operations

| Technology | Why |
|---|---|
| **Docker (multi-stage) / nixpacks** | Deterministic builds and small runtime images; two deployment targets (API, worker) share one build approach. |
| **Railway** | A low-friction PaaS that handles TLS at the edge, manages the MySQL database, and lets the API and web app deploy as separate services with their own domains. |
| **GitHub Actions** | CI/CD close to the code: build and publish the identity package, and run security scanning (Semgrep SAST, GitLeaks) and dependency updates (Dependabot) automatically. |

---

## Design principles behind the choices

1. **One platform owns the cross-cutting concerns.** Identity, session, and email live in one place so
   products stay thin and consistent.
2. **Keep the blast radius small.** The identity library has no email-provider dependency; the API and
   worker are separate deployables; background work (outbox, session cleanup) never blocks a request.
3. **Prefer boring, reliable infrastructure.** Managed database, managed TLS, and a hosted CI all mean
   less custom infrastructure to operate with a small team.
4. **Ship schema as code.** EF Core migrations run at startup, so a deploy is self-contained.
