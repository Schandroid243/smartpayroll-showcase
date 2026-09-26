# Case study — From on-premise software to a production-ready multi-tenant SaaS

**English** · [Français](./CASE-STUDY.fr.md)

## Context

SmartPayroll started as **on-premise** payroll software: a binary shipped to the customer (via `pkg`), a portable MySQL database installed locally, and an offline encrypted license (AES-256-CBC) controlling the active organization and the validity period. That model worked for one customer at a time but couldn't scale: every new customer meant a manual install and manual updates, and the business code physically traveled to the customer's machine (an attack surface for reverse engineering).

The goal became turning this product into a **hosted multi-tenant SaaS platform**, billed through mobile money (the dominant payment method in the DRC, which works as a *push* payment rather than automatic debit). This required rethinking tenant resolution, billing, security and observability from the ground up, not just adding features on top of the existing code.

## The approach: an audit before going to production

Before any real multi-tenant deployment, I ran a full **production-readiness audit** of the project. The question was simple: can this system host several organizations, potentially thousands of users, and be maintained for years?

The audit covered architecture, code, security, data model, performance, tests, CI/CD, observability, frontend, critical business workflows and Git history, with one explicit rule: **never assume that something I couldn't verify (for example, with no real MySQL database available at audit time) was correct.**

### Initial findings

> **Verdict: NOT PRODUCTION READY** for the targeted multi-tenant SaaS.

- **47 technical debt items identified**: 6 critical, 17 major, 19 medium, 5 minor.
- A suite of **663 tests, all green**, but entirely mock-based, and not covering the scenario that actually happens in a multi-tenant environment (organization A reaching organization B's data).

**Top critical findings:**

1. **Multi-tenant isolation not guaranteed**: the active organization was resolved from a global license file, not from the authenticated user's identity; data filters were conditional (`if (req.organizationId)`), so they *failed open* when the value was missing.
2. **HR files exposed without authentication**: the upload folder was served statically and reachable by direct URL, including between employees of the same organization.
3. **Irreversible payroll data loss**: an automatic purge cron cascade-deleted payments after an account had been inactive for 30 days.
4. **Billing not enforced**: the subscription status (`SUSPENDED`, `CANCELED`) had no real effect on access to the application.
5. **Non-reproducible database schema**: `sequelize.sync()` enabled by default at startup, a broken migration chain, and no verified backup strategy.

## Remediation, phase by phase

The remediation plan was split into phases with one simple rule: **never start a phase before the previous one is fully closed.**

| Phase | Goal | Status |
|---|---|---|
| **Phase 0** | Hard blockers before any production release: real tenant isolation (from the JWT, never a client parameter), closing public file access, disabling hard deletes on payroll, enforcing subscriptions, reproducible schema, verified backups | ✅ Done |
| **Phase 1** | Production hardening: health checks, graceful shutdown, non-blocking Redis with DB fallback, CSP/HSTS, distributed rate limiting, structured logs with `requestId`, Node LTS | ✅ Done |
| **Phase 2** | Architecture: a BullMQ worker service separate from the web process, unified organization identifiers (three different identifiers previously coexisted with no foreign key linking them), extraction of dedicated domain services | ✅ Done |
| **Phase 3** | Performance: payroll correction/adjustment workflow, application-level archiving of fast-growing tables | ✅ Done |
| **Phase 4** | Developer experience: full CI (all branches, migrations replayed on an empty database, tests against real MySQL/Redis, client tests, `npm audit`, secret detection, Docker build), gradual re-enabling of strict linting, up-to-date documentation | ✅ Done |

## What a CI that finally runs end to end reveals

The most instructive part of Phase 4 is worth telling in detail, because it shows something rarely discussed: **a green test suite guarantees nothing until it has run under the real conditions it claims to check.**

Before this phase, CI only ran on the main branch, with no real MySQL or Redis. The integration suites meant to run against a real database stayed inert (`describe.skip`) for lack of an environment. Extending CI to every branch and PR with a real MySQL service immediately surfaced, **in a single afternoon**, a series of latent bugs that had never shown up in practice:

- Sequelize migrations older than the reference schema tried to recreate columns and indexes that already existed on a database replayed from scratch. Never an issue in production (the database was never fully rebuilt), but a blocker for any reproducible CI.
- A wrong migration order between two tables linked by a foreign key, caused by misleading lexical sorting of file names (`wallet-` sorted before `wallets`).
- A MySQL session variable (`FOREIGN_KEY_CHECKS`) reset on a different connection from the one used for the next operation, because of the connection pool. Fixed by explicitly pinning a single transaction.
- A Sequelize index referencing the JavaScript attribute name instead of the actual column name. Invisible as long as `sync()` had never rebuilt that schema against a real SQL engine.
- An `INSERT ... SELECT *` between two independently built tables (one via migrations, the other via the Sequelize model) that implicitly assumed the same column order. Fixed with an explicit column list, which also removes the risk of a future regression if a column is added on one side only.
- Test `TRUNCATE TABLE` statements that always failed on tables referenced by a foreign key. This is structural MySQL behavior (blocked as soon as the constraint exists, whatever the table contents), never hit while the corresponding test had never run against a real engine.

Each of these bugs was diagnosed from raw CI logs, fixed, covered by a regression test, then verified by a new CI run, until everything was green.

## Outcome

- Multi-tenant isolation **proven** by an authorization test matrix (organization A / B / none) running against a real MySQL database, rather than by code review or mocks.
- An 8-stage CI pipeline: lint + progressive type checking, unit and integration tests (including against real MySQL/Redis), client tests, dependency audit, secret detection across Git history, production Docker image build.
- Architecture documentation, deployment and backup runbooks, a `CHANGELOG` and a semantic versioning policy kept up to date as the work progressed, not caught up after the fact.
- All 47 technical debt items identified by the initial audit (Phases 0 to 4) are closed.

## What this shows

- **Audit before trusting a green test suite.** 663 passing tests say nothing about what they don't cover.
- **Prioritize by real risk, not by ease.** Multi-tenant isolation (the heaviest piece of work) was handled in Phase 0, before simpler but less critical fixes.
- **Never merge a CI finding you don't understand.** Every bug surfaced by the wider CI was root-caused (not just worked around) and documented with the reason it had never been caught before.
- **A SaaS architecture is designed from the real constraints of its target market**, not from a generic template. The prepaid wallet instead of automatic debit is the most concrete example.

---

← [Back to the README](../README.md)
