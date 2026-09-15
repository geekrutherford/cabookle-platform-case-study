# The Cabookle Products

Cabookle is a small product family for teachers and school librarians. Each product is focused,
opinionated, and free, and all three share one account and one identity system. This document
describes what each product does and the audience it serves.

> All examples and names in this repository are synthetic.

---

## Cabookle Library

**For classroom collections and smaller school libraries.** Library keeps the day-to-day work of a
small library, cataloging, circulation, and helping students find their next book, simple enough
that a teacher can run it without dedicated training or IT support.

### Core capabilities

- **ISBN cataloging.** Scan or type an ISBN and the title, author, cover, and metadata are pulled in
  automatically from book catalogs, no manual data entry.
- **Circulation tracking.** Students check books in and out; staff can see at a glance what's out,
  who has it, how long it's been gone, and who's over their checkout limit.
- **Overdue notices & parent alerts.** Print overdue notices or email parents when books are due;
  reading-history reports support parent-teacher conferences.
- **Interoperability.** Export the collection in formats designed to drop into third-party reading
  tools, so nothing has to be re-entered.
- **AI-assisted discovery.** Students search in plain language and get answers grounded in the books
  the library actually owns (see [AI & natural language](ai-insights.md)).

---

## Cabookle Flow

**For teachers and school librarians organizing the school year.** Flow is a workspace built around
the recurring rhythm of a school year, not a generic task manager.

### Core capabilities

- **Tasks with depth.** Due dates, status, checklists, notes, links, tags, and associated contacts,
  all in one place.
- **Monthly calendar.** A color-coded month view shows every task by due date and category.
- **Templates.** Built-in templates for teacher and librarian workflows that can be copied and
  customized before applying them to a school year.
- **Projects, events & displays.** Log book displays with photos, the books used, and structured
  retrospective notes so each year's displays improve on the last.
- **Contacts.** A directory of vendors, reps, administrators, and colleagues, attachable to tasks.
- **Year-to-year rollover.** Recurring tasks roll forward into the next year with recalculated due
  dates, carrying notes, links, contacts, and tags along.

---

## Cabookle Budget

**For school librarians planning and tracking library spending.** Budget models the real purchasing
cycle, plan, commit, order, receive, report, across a complete budget year.

### Core capabilities

- **Funding sources.** Record every source of money for the year (district funds, grants, book-fair
  proceeds, donations) so the full picture is always available.
- **Categories & allocations.** Split the budget into categories (books, digital resources, supplies,
  programming) and allocate funding to each.
- **Planned purchases.** Capture intent before ordering, with estimated cost, quantity, category,
  priority, and rationale.
- **Vendor orders & line items.** Group planned purchases into orders, track order/payment status,
  and record actual unit costs as line items.
- **Receipts & actual spend.** Log receipts against orders and compare estimated vs. actual costs.
- **Reports.** Funded, allocated, planned, committed, spent, and remaining amounts at a glance,
  scoped to the active budget year.

---

## Cabookle Platform

Every product above is backed by the **Cabookle Platform**, which provides the shared, cross-cutting
capabilities the products don't re-implement themselves:

- **Identity & authentication**: registration, email verification, credential login, and account
  recovery.
- **Single sign-on**: a shared session cookie that lets users move between products without
  re-entering credentials.
- **Organizations & entitlements**: automatic provisioning of an organization and product
  entitlements when a user verifies their account.
- **Transactional email**: verification, password reset, and product-triggered email, all delivered
  through one reliable relay.

The platform is covered in depth in [Architecture](architecture.md),
[Multi-tenancy](multi-tenancy.md), and [Security](security.md).
