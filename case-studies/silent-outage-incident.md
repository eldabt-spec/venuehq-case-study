# Case Study: The Outage That Passed Every Test

**Silent production failure · root-cause analysis · one-line fix · permanent guardrails**

## What happened

In July 2026, the automated jobs that pull new customer inquiries into VenueHQ quietly stopped working in production — and stayed broken for two days. No alarm went off. Every test was green. The system didn't even log an error, because the code was crashing before it got far enough to write one. From the business side, it just looked like customers had stopped inquiring.

## Why nothing caught it

We had added a small, routine library to help clean up email content. Buried inside it was a component written in a newer code format that our hosting provider's servers can't load — even though our own machines handle it fine. That mismatch is why every safety check passed: all of our checks ran on machines where the code works. The only place it failed was the one place we weren't testing — the real servers.

## The process factor: a safety net that wasn't used

There's a second layer to this incident, and it's the more important one. The team had an agreed release process: promotions to production were planned events, run in a written order that ended with health checks on the live system — including probing the exact background jobs that broke. Had that sequence run, the crash would have surfaced within minutes of release as a failed probe, and the fix would have been a same-day rollback instead of a two-day silent outage.

The release that carried this bug went out ahead of that process — promoted by an automated deployment agent before the planned sequence was issued, with no post-release verification at all. So the honest accounting has two lines: the latent bug was ours, and was genuinely undetectable by the checks that existed on developer machines. The two-day outage belonged to the skipped process. A defect that should have been a twenty-minute blip became a multi-day incident not because the code was unusually bad, but because the verification step designed to catch exactly this class of failure never ran.

That distinction shaped the remediation: instead of only fixing the bug, we removed the dependency on anyone — human or agent — choosing to follow the process. The safeguards below run automatically, every time, with no opt-out.

## How we found and fixed it

I traced the outage backward from the release timeline to the exact dependency change, confirmed the incompatible component hidden inside it, and rolled the library back one version to the last release without that component. The fix itself was one line. Before shipping it, we proved it on a live preview of the production environment — actually running the broken jobs and watching them come back healthy — rather than trusting another green build, and we set a hard rule that the release could only go out after the live system visibly recovered.

## What we changed so it can't happen again

An outage that passes every test means the tests were asking the wrong question. Ours asked "does the code build?" when they needed to ask "does the code run where it actually lives?" Two permanent safeguards came out of this:

1. **A pre-merge scan** that detects this exact category of incompatibility before code is ever accepted — catching at review time what previously only surfaced in production.
2. **A post-release health check** that pokes the automated jobs immediately after every deployment and raises a loud failure if they don't respond — so "silent crash" is no longer a possible outcome.

## The takeaway

Modern hosting platforms run code in environments that differ from developers' machines in small but fatal ways. And process discipline is a safety mechanism in its own right: the incident's cost came less from the bug than from the release that skipped verification. This incident taught the team to verify at the level that matters — the running system, not the build — and to make the safety net automatic rather than optional. The fix took one line; the guardrails are the real work.

---

## Technical appendix (for the engineers)

<details>
<summary>Full root-cause chain and remediation details</summary>

**The dependency chain:** a routine addition of `isomorphic-dompurify@2.36.0` (email HTML sanitization) pulled in `jsdom@28`, whose nested dependency `html-encoding-sniffer@6` performs a CommonJS `require()` of the ESM-only package `@exodus/bytes`.

**The trap:** Node 22 supports `require()` of ESM modules — but Vercel's function loader is a separately compiled shim that does not, regardless of the configured Node version. So `vitest` passed (local Node tolerates the require), `tsc` passed (types don't see module formats), local `next build` passed (build-time resolution succeeds), and Vercel's own build passed (the failure is load-time, not build-time). The crash existed only at runtime, inside Vercel's loader, on first invocation — before auth, logging, or any application code ran.

**The fix:** a one-line exact pin to `isomorphic-dompurify@2.26.0` (jsdom 26 tree — no ESM-only transitive requires), shipped with a field-by-field lockfile drift audit proving the pin changed only the intended subtree, adversarial self-review against a 37-item lens taxonomy, and runtime proof: both cron routes exercised on the fix's live preview deployment and returning healthy responses. The production promotion was gated on a staging probe flipping from crash (500) to the expected auth challenge (401).

**The permanent guards:**

- A CI check that requires every server-external dependency under Node's strict CommonJS loader (`--no-experimental-require-module`), reproducing the hosting loader's behavior locally — so the incompatibility fails at PR time.
- A post-deploy smoke script that probes scheduled routes after every production deploy: pass on the expected auth challenge, fail loudly on 5xx, timeouts, or unexpected success.
- A landing-gate policy: any diff touching dependencies or build configuration requires runtime verification on the target platform, not just a green build.

</details>
