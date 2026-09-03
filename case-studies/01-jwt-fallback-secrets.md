# Case Study 1 - JWT Fallback Secrets: Authentication Remediation

**Severity:** Critical
**Surface:** All privileged sessions - CRM team members and platform super-admins
**Outcome:** 4 production files remediated; startup-guard pattern adopted repo-wide; documentation rewritten to match the fixed state

## The finding

VenueHQ runs two separate authentication systems: CRM auth for venue team members (`team_session` cookie) and platform auth for super-admins (`platform_session` cookie). Each signs JWTs with its own secret, read from environment variables.

The problem: multiple production files declared those secrets with a hardcoded fallback:

```typescript
// The vulnerable pattern (reconstructed, not client code)
const JWT_SECRET = process.env.JWT_SECRET || 'some-default-change-in-production';
```

If either environment variable was ever missing - a fresh deployment, a misconfigured preview environment, a variable accidentally deleted from the dashboard - the application would **silently authenticate using a default string that was publicly visible in the source code**. Anyone who had seen the source could forge valid session tokens for the CRM *and* the platform-admin portal, which includes a venue-impersonation endpoint.

The failure mode is the dangerous part: nothing breaks. The app starts, logins work, and the only signal that every session is forgeable is a string comparison nobody is running.

## The fix

Replace silent fallback with a fail-fast startup guard:

```typescript
if (!process.env.JWT_SECRET) {
  throw new Error(
    'JWT_SECRET environment variable is not set. Generate one with: openssl rand -hex 32'
  );
}
const JWT_SECRET = process.env.JWT_SECRET;
```

A missing secret now crashes the deployment at startup - loud, immediate, and impossible to ship - instead of degrading to a forgeable default. Four files carried the pattern: the team-auth library, the platform-auth library, the Edge middleware (which runs on every request and verified both token types), and a venue-impersonation route.

## Verification, not vibes

The remediation wasn't declared done when the four known files were edited. The completion gate was a repo-wide search for the fallback secret strings, restricted to code files, with the requirement that it return **zero matches** - because "I fixed the files I knew about" and "the vulnerability is gone" are different claims, and only the second one matters.

## Adjacent fix: seed-script credential exposure

The same review found the platform-admin seed script accepting the super-admin email and password as command-line arguments - which land in shell history and are visible in process listings while the command runs. The replacement prompts interactively, masks the password, and requires confirmation. Credentials never touch the command line.

## Follow-through

- Rewrote `.env.example` from scratch: all 18 variables documented across 6 setup stages, with security warnings and generation commands (`openssl rand -hex 32`) inline where they're needed
- Updated the README security section to describe the *fixed* behavior (throws on startup) rather than the old one - documentation that contradicts the code is worse than no documentation
- Delivered a deploy checklist and testing guide so the fix could be verified independently in production

## What this illustrates

A `|| 'default'` on a secret is a pattern that looks like defensive programming and is actually the opposite: it converts a loud configuration failure into a silent security failure. The generalizable rule that came out of this: **secrets fail fast; only non-sensitive config gets defaults.**
