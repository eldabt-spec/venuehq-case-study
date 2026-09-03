# Case Study 2 - Multi-Tenant Isolation Audit

**Context:** Platform built single-tenant, being retrofitted to multi-tenant ahead of onboarding a second paying venue
**Outcome:** 6 anonymous cross-tenant leaks found that a prior assessment missed; 30+ routes remediated; ~7 schema-level second-tenant blockers identified across 83 tables; a reusable audit method and leak-pattern taxonomy

## Why isolation was fragile by construction

VenueHQ's server-side code uses the Supabase **service role key**, which bypasses Row Level Security entirely. That means there is no database-level backstop for tenant isolation: every query is responsible for its own `.eq('venue_id', venueId)` filter. A single missing filter is a cross-tenant data leak - the same risk class as a `DELETE` without a `WHERE` clause.

The system had also been built for one venue and retrofitted, so "there's only one venue" assumptions were baked in everywhere.

## Why the existing assessment missed the leaks

A prior structured security assessment had reviewed the codebase and passed the relevant areas. Its method was **static sampling**: read a selection of routes, check them against a rubric. That method has two blind spots for tenant isolation:

1. **Sampling misses the unsampled.** Isolation is only as strong as the *worst* route, so a sampled review answers the wrong question.
2. **Static reading can't see runtime behavior.** A route can look scoped and still leak - e.g., when `venueId` arrives as `null` and a conditional filter silently drops out.

## The method that worked

Two layers, both exhaustive rather than sampled:

**1. Full per-route census.** Enumerate every API route (not a sample), and for each one record: does it read/write venue-scoped tables, where does its venue context come from, and is every query filtered? Grep-driven, then confirmed by reading each flagged route.

**2. Runtime probing against a disposable second tenant.** Create a second venue in the staging database, then probe routes as an anonymous/unprivileged caller and check whether tenant A's data is reachable from tenant B's context - or from no context at all. This is what static review structurally cannot do.

The runtime layer is what surfaced **6 anonymous public cross-tenant leaks** the static assessment had missed.

## The leak-pattern taxonomy

The audit distilled recurring shapes, now documented in the project's architecture docs so they never get re-introduced:

**1. The "first venue in DB" fallback.** When venue context is missing on a public request, the code picks the first active venue instead of rejecting. Any entry point lacking the `?venue=<slug>` parameter bleeds into whichever venue happens to sort first. Rule: never silently default to a tenant - reject, log, surface.

**2. Hardcoded tenant slugs.** `.eq('slug', '<venue>')` in cron code meant every scheduled job ran only against the original tenant; a second venue would silently get no automation at all.

**3. "Start unscoped, conditionally add the filter."**
```typescript
let query = supabase.from('packages').select('*');
if (venueId) query = query.eq('venue_id', venueId);  // null venueId → all tenants' rows
```
Rule: start scoped or throw - `if (!venueId) throw`, then build the query.

**4. Global UNIQUE constraints on venue-scoped tables.** An audit of 83 multi-tenant tables found ~7 constraints (user emails, team-member emails, sequential quote numbers, package names) that were globally unique instead of unique-per-venue. Dormant with one tenant; the day a second venue's customer reuses an email, the insert throws `23505 duplicate key` and surfaces as a 500 on a public form. Rule: composite uniqueness - `UNIQUE (venue_id, email)`.

**5. Missing `venue_id` on INSERT.** Reads correctly scoped, writes orphaned - rows exist but no tenant's UI can see them.

**6. Shared external infrastructure with per-venue iteration.** The root cause of a 6-week production incident: an email-ingestion cron iterated venues correctly in code, but the Gmail mailbox it read from was configured by a single environment-wide variable. One venue's emails were ingested and auto-replied to under another venue's identity - 16 leads misattributed. Rule: external integrations (mailboxes, webhooks) must be configured per-tenant in the database, never via shared env vars.

## Remediation shape

- Access-control gaps closed across 30+ API routes (IDOR audit + scoping fixes)
- Public API changed from "fall back to first venue" to **hard-fail without venue resolution**, with the embedding site updated to always pass the venue parameter - verified as a promotion-day checklist item, since the change converts silent misrouting into a visible failure if any embed is missed
- The hardcoded-tenant cron fallback removed entirely: every scheduled job now resolves its venue explicitly, and jobs fail closed when their secret is unset rather than running unauthenticated
- Anonymous row-level-security boundaries closed across 20+ tables - `USING(true)` policies scoped to the row's venue, `SECURITY DEFINER` execute permissions revoked from the anonymous role, and search paths pinned so anonymous callers can no longer read cross-venue data or provision a tenant
- Schema fixes staged behind a guarded backfill: constraints and `NOT NULL` deferred until production data is verified, with the backfill guarded to run only when exactly the expected tenant set exists
- Known residual risks (service-role RLS bypass, join/RPC scoping gaps) documented as tracked limitations rather than left implicit

## What this illustrates

The meta-finding matters more than any single leak: **a security assessment's method determines what it can see.** Static sampling produced a false "pass" on the exact property the business most needed. The fix was not "look harder" - it was changing the method to exhaustive census plus runtime probing, then encoding the findings as patterns so the review scales beyond one person's memory.
