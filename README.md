# VenueHQ - Technical Due Diligence & AI QA Case Study

**Role:** Technical Advisor & AI QA Lead (March 2026 – present)
**Stack:** Next.js (App Router) · TypeScript · Supabase (Postgres) · Vercel
**Codebase:** Private multi-tenant SaaS venue-management platform (2,000+ PRs on the integration branch). This repo documents my work on it without exposing client code.

---

## What VenueHQ is

A multi-tenant SaaS platform for event-venue businesses: public pricing calculator and inquiry forms, a CRM for venue teams (quotes, bookings, packages, finance), a platform-admin portal, automated email follow-ups, and scheduled cron jobs - all serving multiple venue tenants from one deployment.

I was brought in to assess production readiness and lead QA as the platform prepared to onboard its second tenant. The engagement expanded into security remediation, multi-tenant isolation auditing, migration planning, CI enforcement, and building the AI-assisted QA process the project now runs on. Over the course of it the platform moved from a static readiness assessment through a sustained hardening push, a clean production promotion, and into a phase where autonomous AI coding agents do much of the implementation work - with me as the independent human verification layer over them.

## Headline results

| Area | Outcome |
|---|---|
| **Authentication** | Found and remediated hardcoded JWT fallback secrets in 4 production files - a vulnerability allowing anyone with source access to forge valid sessions for both the CRM and platform-admin portals |
| **Full-surface authorization census** | Censused all 454 API routes for tenant scoping, confirming 6 anonymous public cross-tenant leaks and 6 authenticated CRM IDOR vulnerabilities a prior static assessment had missed |
| **Guest-portal cross-tenant leak (shipped to prod)** | Proved a live production vulnerability - a guest token from one venue plus a caller-controlled tenant header returned another venue's quote and payment data on 4 routes - via a two-venue probe kit; TDD fix flipped the probes 200→403/401, 66/66 tests green, promoted to production with a read-only prod safety precheck confirming zero exploited rows |
| **Database-layer privilege escalation (fixed in prod)** | Autonomous audit of `SECURITY DEFINER` functions found anonymous callers could escalate to admin via 13 functions missing search-path pins; remediated directly in production via a transactional `ALTER FUNCTION` block after read-only verification |
| **RLS remediation** | Discovered a policy-recursion error was accidentally the only thing preventing anonymous reads of every venue's admin role structure; shipped the recursion fix and exposure closure as one unsplittable migration package, verified by an all-zeros anonymous probe with zero member-visibility regression |
| **Production incident: database CPU crisis** | Root-caused 19 days of 100% CPU, ~470 failed transactions/sec, and 38.8M Postgres errors/day to a migration raising business conflicts as SQLSTATE 40001, auto-retried by PostgREST; fix confirmed within 30 seconds of deploy (rollbacks 470/s → 0, errors 1.67M/hr → 9/hr), followed by a sweep migration eliminating the retry class across all database functions plus a CI ratchet banning new occurrences |
| **Performance forensics** | Root-caused a notification-scheduler CPU incident to a timezone function rescanning 1,196 `pg_timezone_names` rows per venue per page; fix achieved a 24× speedup (771ms → 33ms) with a mutation-tested equivalence suite |
| **Silent-failure discovery** | Overnight log analysis found the public inquiry form fully broken for ~35.5 hours (hotfixed same day) and 2 CRM settings tabs throwing 500s for every venue for ~6.5 months due to tables dropped out-of-band |
| **AI agent oversight** | Sequenced a 14-migration promote delta (13 authored by an autonomous agent) against staging and prod ledgers; separately intercepted an agent's mid-promote request to apply 27 prod schema changes after an independent diff found a 32-vs-30 migration file discrepancy and an unauthorized self-merge claim |
| **Release management** | Executed the full 319-commit staging→main promotion ending 6 weeks of environment divergence, with bucket-by-bucket rulings on 306 agent-authored commits and zero-downtime deploy; authored the 5-step "schema before code, data-fix after code" promote runbook the team adopted |
| **Multi-tenant isolation** | Systematic per-route `venue_id` audit + runtime probing against a disposable second tenant; closed access-control gaps across 30+ API routes |
| **Schema audit** | Audited 83 multi-tenant tables; found ~7 global UNIQUE constraints (emails, quote numbers) that would break the second tenant's inserts with `23505` errors on day one |
| **Error/XSS hardening at scale** | Landed an error-detail leak sweep with a static lint guard verified at 0 violations across 492 API routes, plus AI-generated-HTML sanitization and email-template escaping verified against hostile payloads on staging |
| **Concurrency & data-integrity audits** | Multiuser concurrency audit found zero realtime updates product-wide, silent last-write-wins on task fields, a concurrent notes-append data loss, and a view-only-role privilege escalation; money-value sweep fixed 19 sites where fallback values silently mispriced customer totals |
| **CI enforcement & cost** | Made lint, type-check, and accessibility contrast genuinely blocking; later cut the promote CI gate from ~16 to ~6-8 minutes, reduced test-suite wall time 38.6% (962-file audit, zero tests lost), and shipped an automated post-deploy production probe that caught 2 real deployment misconfigurations in live-fire testing |
| **Environment config** | Audited 18 production environment variables against the live Vercel deployment; surfaced 2 confirmed gaps before launch |
| **Incident work** | Root-caused a 6-week cross-tenant email leak (shared Gmail impersonation env var vs. per-venue config) and specified the per-venue fix |
| **Velocity with gates** | ~210 PRs merged to the integration branch in a two-week window, and dozens of unattended overnight agent runs since - every change through adversarial review, with a 5,700+ test suite kept green throughout |
| **Process** | Built the AI QA workflow the project runs on: a 37-lens adversarial review rebuilt from a 346-finding audit of the complete AI reviewer corpus, calibrated confidence labels, four-question investigation gates, and premise-gated autonomous batch runs with STOP-AND-REPORT gates and alive-signal monitoring |

## Case studies

1. **[JWT fallback secrets - authentication remediation](case-studies/01-jwt-fallback-secrets.md)**
   How a "sensible default" became a session-forgery vulnerability, and the startup-guard pattern that fixed it.

2. **[Multi-tenant isolation audit](case-studies/02-multi-tenant-isolation-audit.md)**
   Why the existing static security assessment couldn't catch cross-tenant leaks, the census + runtime-probe method that did, and the leak pattern taxonomy that came out of it.

3. **[Environment variable audit](case-studies/03-environment-audit.md)**
   Cross-referencing documented config against the live deployment, and rebuilding `.env.example` and the README so the docs match reality.

4. **[Production-verified findings: proving exploits before fixing them](case-studies/04-production-verified-findings.md)**
   Three findings that reached "proven with a live probe" before any fix was designed - a guest-portal cross-tenant leak, a database-function privilege escalation, and an RLS policy whose *bug* was its only safety property - and how each was verified, fixed, and re-verified in production.

5. **[Turning CI from theater into a gate](case-studies/05-ci-enforcement.md)**
   A CI pipeline that reported green while real violations accumulated behind `continue-on-error`, and the incremental campaign that made type-checking, lint families, and accessibility contrast genuinely blocking without halting a fast-moving team.

6. **[The Outage That Passed Every Test](case-studies/silent-outage-incident.md)** Root-causing a silent production failure invisible to every CI gate

## Process documentation

- **[AI QA workflow](process/ai-qa-workflow.md)** - how I run QA with Claude Code as the execution layer and myself as the verification layer: evidence labels (Verified / Inferred / Hypothesized), the four-question investigation gate, adversarial review, and autonomous-run architecture with STOP-AND-REPORT gates.

## What this demonstrates

- Security auditing of a production SaaS system (authn, authz, tenant isolation, config)
- Independent human oversight of autonomous AI coding agents: verifying their claims against live systems, catching discrepancies and unauthorized actions mid-flight
- Root-cause discipline: no fix designed until the cause is *Verified*, the surface area is measured, and production state is confirmed
- Production incident response with quantified before/after outcomes
- AI-assisted engineering with real quality gates - treating LLM output as untrusted until grounded in code
- Clear technical writing: every finding here was originally delivered to stakeholders as an actionable document

---

*All examples are described at the pattern level. No proprietary client code appears in this repo. Identifying details of tenant businesses have been generalized.*
