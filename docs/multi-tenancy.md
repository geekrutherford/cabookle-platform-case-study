# Multi-Tenancy & Data Isolation

Cabookle serves many independent organizations — typically a school or a teacher running their own
library — inside a single shared application. This document explains the tenancy model and how one
tenant's data is kept isolated from another's.

> This is a conceptual description of the model. It is intentionally **not** a complete schema.

---

## The tenancy model

Cabookle uses a **shared-database, shared-schema** model where every row is scoped to an
**organization** (the tenant). The platform holds the tenant graph:

```mermaid
erDiagram
    ORGANIZATION ||--o{ ORGANIZATION_MEMBERSHIP : "has"
    PLATFORM_USER ||--o{ ORGANIZATION_MEMBERSHIP : "belongs to"
    ORGANIZATION ||--o{ PRODUCT_ENTITLEMENT : "is entitled to"
    PRODUCT ||--o{ PRODUCT_ENTITLEMENT : "grants access to"

    ORGANIZATION {
        string name
        string slug
        bool is_active
    }
    PLATFORM_USER {
        string email
        string first_name
        string last_name
        bool is_active
        bool email_verified
    }
    ORGANIZATION_MEMBERSHIP {
        role
        bool is_active
        bool is_default
    }
    PRODUCT {
        string code
        string name
        bool is_active
    }
    PRODUCT_ENTITLEMENT {
        bool is_active
        datetime expires_at
    }
```

### Key entities (conceptual)

- **Organization** — the tenant. Created automatically when the first user verifies their email.
- **PlatformUser** — a person. A user can belong to more than one organization.
- **OrganizationMembership** — the join between a user and an organization, carrying a **role**
  (e.g. owner/admin vs. staff) and an "is default" flag for the user's primary organization.
- **Product** — a fixed catalog of the Cabookle products (`library`, `flow`, `budget`).
- **ProductEntitlement** — grants an organization access to a product, optionally with an expiry and
  per-organization settings.

### Product-scoped tenancy

Within each product, the organization remains the isolation boundary. For example, a Library tenant
further owns its collection, students, and circulation records — all ultimately rooted in one
organization id. Flow scopes everything to a school year *within* an organization, and Budget scopes
everything to a budget year *within* an organization.

---

## How isolation is enforced

The isolation strategy rests on a small number of invariants:

1. **Tenant identity is established once, at the platform.** When a user signs in, the platform issues
   a short-lived token that carries the user's **organization id**, role, and entitled product codes.
   Downstream services trust these claims rather than re-deriving them.

2. **Every tenant-scoped query is filtered by organization id.** The critical rule is: no query that
   returns tenant data is written without an explicit organization filter. A product's own data is
   always reachable through its owning organization.

3. **Entitlement is checked, not assumed.** Access to a product is mediated by an active entitlement
   for that organization; the token advertises which products the user is entitled to, and services
   enforce it.

4. **Cross-tenant access is prevented by construction.** Because identifiers are always resolved
   through the tenant's own graph (user → membership → organization → product data), there is no
   lookup path that can traverse from one tenant's records into another's.

### Illustrative invariant (see [code samples](code-samples.md))

The single most important pattern is the **tenant-scoped query guard** — a data-access helper that
forces every read to be constrained to the caller's organization. A simplified version appears in the
code samples; the real implementation is the same idea applied consistently across every product.

---

## Why shared-database (rather than per-tenant databases)

For a product aimed at many small schools with a small team operating it, **a single database with
strict tenant scoping** keeps operational complexity low while still providing isolation:

- One schema to migrate, one backup to take, one connection pool to tune.
- New tenants are just rows, so self-service signup is trivial (no provisioning a database per signup).
- Isolation is guaranteed in the application layer (scoping + entitlements) rather than by
  infrastructure, which is the right trade-off at this scale.

The cost is that isolation discipline must be upheld in code — which is why the tenant-scoped query
guard is treated as a first-class invariant, not a convention.
