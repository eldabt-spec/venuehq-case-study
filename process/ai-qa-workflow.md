# The AI QA Workflow

How VenueHQ development runs with an AI coding agent (Claude Code) as the execution layer and a human QA lead as the verification layer. The core stance: **AI output is untrusted until grounded in the actual code.** Everything below exists to enforce that stance mechanically instead of relying on vigilance.

## 1. Calibrated confidence labels

Every factual or diagnostic claim - in PR bodies, issue comments, handoff docs - carries one of three labels:

- **Verified** - the code/config/output was actually read or run
- **Inferred** - reasoned from symptoms; plausible but not confirmed
- **Hypothesized** - a theory, unchecked

The label travels with the claim. Hypotheses never get filed as facts, and a claim that can't earn a label is answered with "I'd need to check." This vocabulary was adopted after catching AI-generated artifacts asserting fabricated UI navigation paths, invented feature keys, and "completed" claims for work that hadn't run.

## 2. The four-question investigation gate

No fix gets designed until the investigation answers four questions - separately for every environment the fix will touch (local, staging, production):

1. **Root cause** - labeled *Verified*, not *Hypothesized*
2. **Verified-working version** - evidence (diff, test output, screenshot) that a candidate fix produces the working state, or an explicit statement that none has been tested yet plus a plan to test one first
3. **Full surface area** - measured up front by grep, lint, or count queries. "We'll find the rest during implementation" means the investigation isn't done
4. **Production-state precheck** - for any bug found in local QA, one read-only check confirming the bug exists in production. If it doesn't, the issue is re-scoped to environment parity before anyone designs anything

If the answers differ across environments, that difference *is* part of the root cause. "Works in prod but not local" is a different problem than "broken everywhere."

## 3. Adversarial review

Every PR-sized unit of work passes a structured adversarial self-review before landing: a 37-lens checklist distilled from the complete corpus of external code-review findings on the project. Each lens is a named failure class (tenant-scoping gaps, silent fallbacks, duplicate-send races, cache-header bleed, migration ordering, and so on) with a concrete check. The review produces a findings file per unit; a landing prompt without review evidence attached is by definition not ready.

This runs alongside multi-AI cross-checking - independent AI review bots on every PR, with a documented accept/reject decision and reasoning per suggestion. Disagreement between reviewers is signal, not noise.

**Evidence citation rule.** Every line of a pre-work checklist must cite the exact query that was run plus a named artifact - a PR number, branch name, session date, or `file:line`. A checklist line without one is invalid, regardless of how confident it sounds. This rule exists because the most common process failure isn't a wrong answer, it's a *session-level* search presented as if it were a *per-item* search: one broad query run once, then cited on eight separate checklist lines that were never individually verified.

## 4. Prompt engineering as reliability engineering

Prompts to the coding agent are pre-flighted against a checklist before they run:

- **Incremental writes** - long runs write progress to disk after each section, so a stream timeout doesn't lose the work
- **STOP-AND-REPORT gates** - any step whose correct action depends on an earlier step's output halts and reports instead of continuing blindly
- **Worked examples** - any non-obvious syntax (shell escaping, heredocs, quoted SQL identifiers) gets one worked example in the prompt, because this failure mode is silent: it looks fine until the next step breaks
- **Destructive-op confirmation** - deletes, resets, force-pushes require explicit confirmation
- **Cancel thresholds, not duration estimates** - "if still running at N minutes with no report, interrupt" is actionable; "should take 2–4 minutes" is uncalibrated guessing
- **After any failure, smaller is correct** - a timed-out prompt gets split, never expanded

## 5. Autonomous batch runs

Unattended overnight work runs under a constrained architecture:

- One branch per work item, cut fresh from the integration branch
- **Local commits only** - no pushes, no PRs, no database contact from autonomous runs
- Counted retry budgets with a SKIP-and-log fallback, so one stuck item can't consume the run
- Per-item digest and adversarial-review files; a final return-report written before exit
- State-bounded kill switches (the run stops when a defined state is reached), never wall-clock self-policing - an agent asked to watch its own clock will hallucinate having done so
- A mandatory "you will NOT be re-invoked" clause, after repeated observations of the agent inventing a fictional future session that would finish the gates it skipped

**Premise gates.** Every queued item opens by testing its own premise against the live codebase before doing any work, and exits with a SKIP if the premise is false. This was added after items were queued from stale backlog descriptions and the run dutifully "fixed" things that were already fixed - in one case a tracked issue whose fix had landed days earlier under an unrelated PR. Two of six items in a recent run exited on premise gates. That is the gate working, not the run failing.

**Landing status is verified by content, not by git metadata.** Because the project squash-merges, `git log`, `git cherry`, `git branch --no-merged`, and raw diffs *all* falsely report landed branches as unlanded - squashing rewrites patch identity while preserving content. Even the merged-PR list is unreliable when a branch merges under a sibling name. The only signal that holds is per-file blob-SHA comparison between the branch and the integration branch. A census using this method resolved an apparent nine-item backlog down to one genuinely outstanding item; the rest were already shipped and the records were wrong.

**Landing is serialized, not batched.** Merges run one at a time - wait for checks, then merge, then next - which additionally dissolves same-file conflicts between queued branches without manual intervention.

Security-sensitive and migration work is explicitly excluded from autonomous runs and done attended.

## 6. Verification workflow rules that earned their place

Each of these encodes a real observed failure:

- **A production build is part of the landing gate** whenever a diff touches dependencies or build config, or adds the first server-side import of a client-targeting library - after a case where tests and type-checking both passed but the deployment failed at runtime on a library that couldn't resolve its assets server-side
- **Type-check gates diff against a clean baseline worktree**, not against "the tail of the output looks fine" - with ~210 known pre-existing errors, only a diff distinguishes "you added errors" from "the noise floor"
- **Test-count baselines are recorded** (files and test counts) so "tests pass" can be distinguished from "tests silently didn't run"
- **Migrations are authored by AI, applied by humans** - the agent writes migration files; a human applies them through the database console, one statement at a time with expected output stated in advance
- **CI gates get audited too.** A full read of every workflow file found that several lint gates ran with `continue-on-error: true` - they reported green while real violations accumulated behind them - and that CI had no type-check, lint, or build step at all, meaning the deploy platform was silently load-bearing for correctness. A passing pipeline is a claim like any other; it needs verification before it earns trust
- **Regressions get a permanent automated gate, not a note.** A UI batch worsened text-on-fill contrast from 3.02:1 to 2.15:1, below the 3.0 non-text accessibility floor. The fix shipped with an automated contrast-verification check wired into CI, so the specific regression class cannot recur silently. The general rule: when a class of defect escapes review once, the remediation includes the mechanism that catches it next time

## The through-line

None of this is anti-AI process. The agent does the majority of the mechanical work, and does it fast. The process exists because the failure modes of AI-assisted development are *silent by default* - fabricated paths look like real paths, skipped gates look like passed gates. Every rule above converts a silent failure into a loud one.
