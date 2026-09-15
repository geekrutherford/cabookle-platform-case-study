# Monitoring & Production Support

This document describes how the Cabookle platform is operated: what's observable, how problems are
detected and diagnosed, and the production-support practices.

---

## Observability

The platform keeps observability deliberately simple for a small team:

| Signal | Mechanism | Why |
|---|---|---|
| **Liveness** | A lightweight `/health` endpoint returns service status. | Load balancers and uptime checks can verify the service is up without hitting a real feature. |
| **Structured logging** | The API and worker emit structured logs (with context like the failing operation and exception) rather than free-form strings. | Searchable, greppable diagnostics when a background job or request fails. |
| **Deployment events** | Startup logs confirm that migrations ran and that seeding completed. | A deploy's most fragile step (schema change) is explicitly observable. |
| **Background job logging** | The session-cleanup and email-outbox jobs log each run's outcome and any errors. | Long-running work that would otherwise be invisible is surfaced. |

### What's watched

- **Email delivery health.** The outbox retries with backoff and records the last error per message, so
  a provider outage is visible as a growing backlog of retrying messages rather than silent loss.
- **Session & account behavior.** Rate limiting and lockout metrics indicate credential-stuffing or
  abuse attempts.
- **Startup failures.** A failed migration stops the service loudly (fail fast) instead of running
  against a wrong schema.

---

## Detecting and diagnosing problems

1. **Is it up?** The health endpoint answers immediately.
2. **Is it a deploy problem?** Startup logs show whether migrations ran and completed.
3. **Is it a background problem?** The worker's structured logs show per-run outcomes and per-message
   errors (with bounded retries already in flight).
4. **Is it abuse?** Rate-limit/lockout signals distinguish a real outage from a spike of automated
   traffic.

---

## Production-support approach

- **Fail fast on misconfiguration.** Required settings (JWT key, email key) are validated at startup;
  the service refuses to start with a clear message instead of failing at the first request.
- **Background work is decoupled.** The worker is a separate deployable, so a stuck outbox or cleanup
  job can be restarted or scaled without touching the request path.
- **Emergency toggles.** Security controls (for example, bot verification) can be disabled via
  configuration in an emergency without a redeploy.
- **Self-contained deploys.** Migrations run at startup, so rolling out a release is a single,
  reviewable action.
- **Independent key rotation.** Each service has its own internal API key, so a single compromised
  credential can be rotated without a coordinated multi-service change.

### Runbook philosophy

For a small team, the operating model favors **few moving parts and loud failures** over a complex
observability stack: every background job logs its outcome, every deploy logs its migration step, and
every external dependency has an explicit failure path (retry, backoff, and a visible error).
