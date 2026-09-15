# Architecture

This document describes the high-level architecture of the Cabookle suite: the system context, the
components of the platform, and how the products consume it.

> Diagrams are authored in Mermaid. The same sources live in [`diagrams/`](../diagrams/).

---

## 1. System context

```mermaid
flowchart LR
    subgraph People
        LIBRARIAN["School librarians"]
        TEACHER["Teachers"]
        STUDENT["Students"]
    end

    subgraph Products["Cabookle products"]
        LIB["Library"]
        FLOW["Flow"]
        BUD["Budget"]
    end

    subgraph Platform["Cabookle Platform"]
        API["Platform API"]
        SPA["Platform web app"]
        WORKER["Background worker"]
        DB[("MySQL")]
    end

    subgraph External["External services"]
        EMAIL["Email provider"]
        CAPTCHA["Bot protection (reCAPTCHA)"]
        CATALOG["Book metadata catalog"]
    end

    LIBRARIAN --> LIB
    LIBRARIAN --> FLOW
    LIBRARIAN --> BUD
    TEACHER --> LIB
    TEACHER --> FLOW
    STUDENT --> LIB

    LIB --> API
    FLOW --> API
    BUD --> API
    SPA --> API

    API --> DB
    WORKER --> DB
    API --> EMAIL
    API --> CAPTCHA
    LIB --> CATALOG
```

### What it shows

- **Products are separate applications** that all call the same platform API for identity, session,
  and email.
- **The platform is the single source of truth** for users, organizations, and entitlements.
- **External dependencies are narrow** — one email provider, one bot-protection provider, and a book
  metadata catalog used only by Library.

---

## 2. Platform components

```mermaid
flowchart TB
    subgraph Platform["Cabookle Platform"]
        direction TB
        API["Platform API<br/>(ASP.NET Core)"]
        WEB["Platform Web<br/>(Svelte 5 SPA)"]
        WORKER["Background Worker<br/>(.NET Worker)"]
        IDENTITY["Identity library<br/>(EF Core entities & services)"]
        DB[("MySQL")]
    end

    WEB -->|"HTTP (JWT / session cookie)"| API
    API --> IDENTITY
    WORKER --> IDENTITY
    IDENTITY --> DB
```

### Responsibilities

| Component | Responsibility |
|---|---|
| **Platform API** | Auth endpoints (register, verify, login, logout, session, me), admin endpoints, contact form, and the internal email relay. Owns middleware for security headers, rate limiting, and internal API-key validation. |
| **Platform Web** | The user-facing single-page app for login, registration, dashboard, and account management. |
| **Background Worker** | Runs the email outbox: picks up queued messages and delivers them with retries, separate from the request path. |
| **Identity library** | A reusable class library with EF Core entities, the `DbContext`, and identity services (password hashing, tokens, sessions). Consumed by the API and Worker, and published as a package for other services. |
| **MySQL** | Relational store for users, organizations, memberships, entitlements, sessions, and the email outbox. |

The separation of **Identity** into its own library is deliberate: it keeps identity logic free of
hosting concerns and lets multiple processes (API + Worker) share one implementation without
duplication.

---

## 3. How a product consumes the platform

```mermaid
sequenceDiagram
    autonumber
    participant U as User (browser)
    participant P as Product (e.g. Library)
    participant API as Platform API
    participant DB as MySQL
    participant M as Email provider

    U->>P: Open product
    P->>API: GET /session (cookie sent automatically)
    API->>DB: Validate server-side session
    alt valid session
        API-->>P: Short-lived JWT
        P-->>U: Signed-in experience
    else no/invalid session
        P->>P: Redirect to platform login
    end

    Note over P,API: Later: product needs to send email
    P->>API: POST /email/send (X-Internal-Key)
    API->>DB: Enqueue outbox message
    API-->>P: 202 Accepted
    Note over DB,M: Worker delivers asynchronously
    DB-->>M: Deliver + retry on failure
```

Key ideas:

- **No token in the URL.** The browser's HttpOnly cookie is sent automatically to the shared domain,
  and the platform exchanges it for a short-lived JWT.
- **Email is fire-and-forget for the caller.** Products enqueue; the platform owns reliable delivery.
