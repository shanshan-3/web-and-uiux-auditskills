---
name: production-audit
description: |
  Audits a web project for production readiness: legal compliance, analytics, error monitoring, infrastructure hardening, database operations, and authentication flows. Triggers on: "production audit", "launch checklist", "ready to deploy", "pre-launch check", "go-live audit", "production readiness". Use when preparing to ship a website or web app to production. Also owns the checks for secrets in client code, HTTPS enforcement, and security headers.
disable-model-invocation: true
license: MIT
---

Scan the project for production readiness. For every item in each category below, determine its status and output a report. Read source files, config files, environment schemas, and public assets to verify. Do not assume.

## Steps

1. Identify the project root and the stack. Locate `package.json`, `composer.json` or `artisan` (Laravel), the framework config, and the deployment config (`vercel.json`, `netlify.toml`, `Dockerfile`, and so on). Note the database and auth provider in use (Supabase, Prisma, Eloquent, Auth.js, Laravel Breeze or Fortify).
2. Run each checklist category. For each item: read the relevant file(s), determine status, record result.
3. Output the full report as a markdown table per category (see format below).
4. After every table, list each Missing and Partial item with a severity and a concrete fix. See [fixes.md](references/fixes.md) for fix templates.
5. End the report with a "Manual checks" list covering every item marked Manual.
6. Done when every item has a status and every gap has a named fix.

## Rules

- Detect the stack first. If an item depends on a tool the project does not use (for example Supabase, Sentry, payments), mark it N/A and give a one-line reason. Never mark Missing for tooling the project does not need.
- If an item cannot be verified from source (dashboard settings, hosting configuration, backups, uptime monitors, git history), mark it Manual and state exactly what the user should check. Never guess Applied.
- Give every Missing and Partial item a severity: Critical (security, data loss, legal exposure), High (user-facing breakage, launch blockers), Low (polish). Sort the fix list by severity, Critical first.
- Cite file and line for every Applied, Missing, and Partial result.
- Do not use emojis in the report.

## Report Format

```
### Category Name
| Item | Status | File / Evidence |
|---|---|---|
| Privacy policy route | Applied | app/privacy/page.tsx |
| Cookie consent banner | Missing | No consent library or banner component found |
| Sentry SDK configured | Partial | Client-side only, no server instrumentation |
| Database backups | Manual | Confirm backups or PITR in the database dashboard |
| Payment idempotency keys | N/A | No payment flow in this project |
```

Then output the fix list:

```
### Fixes (sorted by severity)
1. [Critical] Passwords hashed - app/api/signup/route.ts:31 stores the raw password. Hash with bcrypt (see fixes.md, Auth).
2. [High] Cookie consent banner - add a consent component and gate analytics on it.
```

Status legend: Applied, Missing, Partial, N/A, Manual.

---

## Checklist

### 1. Legal & Privacy
| Item | What to check |
|---|---|
| Privacy policy page | Route exists (`/privacy` or `/privacy-policy`) with substantive content, not a placeholder |
| Terms of service page | Route exists (`/terms` or `/terms-of-service`) with substantive content |
| Footer links to legal pages | Privacy and Terms linked from the site footer or signup flow |
| Cookie consent banner | Consent banner present; non-essential tracking scripts (analytics, ads) blocked until user accepts. Mark N/A only if the site sets no non-essential cookies and loads no tracking scripts |
| Cookie consent persisted | Consent choice saved to a cookie or `localStorage`; banner does not reappear on every visit |

Scan: `app/` or `pages/` route files, footer/layout components, cookie consent library imports (`cookie-consent`, `react-cookie-consent`, `cookieyes`). For Laravel, scan `routes/web.php` and Blade layouts.

### 2. Analytics & Monitoring
| Item | What to check |
|---|---|
| Website analytics configured | Analytics script installed and initialized (GA4, Plausible, PostHog, Vercel Analytics, or Umami) |
| Analytics respects cookie consent | Analytics script loads only after user consent (if a cookie banner is present and the analytics tool uses cookies) |
| Error monitoring (client) | Error boundary and monitoring SDK on the client (Sentry, LogRocket, Bugsnag, or equivalent) |
| Error monitoring (server) | Server-side error capture configured in API routes or middleware (Sentry `init` on the server, or structured error logging) |
| Uptime monitoring | External uptime check configured (UptimeRobot, BetterStack, Checkly). Always Manual unless a config file for it is in the repo |

Scan: `app/layout.*`, `_app.*`, `instrumentation.ts`, `global-error.*`, `sentry.client.config.*`, `sentry.server.config.*`, analytics provider imports, middleware files.

### 3. Infrastructure & HTTPS
| Item | What to check |
|---|---|
| HTTPS enforced | Deployment config or app code redirects HTTP to HTTPS; no mixed-content `http://` asset URLs in source (ignore XML namespace URLs such as `http://www.w3.org/2000/svg`). Hosting dashboards that redirect automatically are Manual |
| Security headers configured | `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY` (or CSP `frame-ancestors`), `Referrer-Policy`, and `Content-Security-Policy` set via middleware, `next.config`, or hosting config. A CSP that allows `'unsafe-inline'` or `'unsafe-eval'` scripts is Partial |
| Secrets not exposed in client code | No API keys, secrets, or service-role keys hardcoded in client-side code; secrets read from environment variables server-side only; no secret uses a client-exposed prefix (`NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`) |
| `.env` ignored by git | `.env` and `.env.local` are in `.gitignore` and not committed |
| Secrets absent from git history | Cannot be fully checked from the working tree. Mark Manual and recommend running `npx gitleaks detect` |
| `.env.example` present | An `.env.example` or `.env.template` exists listing required variables without real values |

Scan: `next.config.*`, `vercel.json`, `netlify.toml`, middleware files, `headers()` config, `.gitignore`, `.env*` files, client source for `sk_live`, `service_role`, `SECRET`, `PRIVATE`.

### 4. Database & Backend Operations
| Item | What to check |
|---|---|
| Database backups configured | Backup strategy documented or configured (Supabase PITR, managed Postgres backups, or cron-based `pg_dump`). Mark Manual if not in code, and ask the user to confirm a restore has been tested |
| Connection pooling | Database connections use a pooler (Supabase Supavisor, PgBouncer, or Prisma connection pooling), not direct connections in serverless. Mark N/A for a single long-running server |
| Safe account deletion | User deletion soft-deletes or cascades without breaking referential integrity; foreign key constraints or `ON DELETE` rules defined |
| Payment idempotency keys | Payment and transaction submissions include a client-generated idempotency key that is reused on retry, with a unique constraint server-side. Mark N/A if there is no payment flow |
| Private storage buckets | File storage buckets (Supabase Storage, S3, R2, Laravel disks) are not publicly listable; access controlled via signed URLs or RLS policies. Mark N/A if the project stores no files |

Scan: database migration files, schema files, Supabase config, storage bucket policies, payment API route files, user deletion handlers.

### 5. Auth & Critical Flows
| Item | What to check |
|---|---|
| Signup flow complete | Registration validates email format, enforces a minimum password length (12 recommended) and rejects known-breached or common passwords, and handles duplicate emails with a neutral message |
| Login flow complete | Login returns a structured error on invalid credentials without revealing whether the email exists; rate-limited |
| Password reset flow complete | Reset sends an email with a time-limited, single-use token; reset page validates token expiry; success redirects to login |
| Post-auth redirect | After login or signup, user is redirected to their intended destination (not always the homepage), and the redirect target is validated to be a same-site path (no open redirect) |
| Session expiry handled | Expired sessions redirect to login with a message; no silent failures or blank screens |
| Password hashing | Passwords hashed with bcrypt, scrypt, or Argon2, never stored in plaintext or with MD5/SHA1. Mark N/A only if authentication is fully delegated to a third-party provider |

Scan: auth route files, middleware, session config, password handling in signup/login handlers, Supabase auth config. For Laravel, scan `app/Http/Controllers/Auth`, `routes/auth.php`, and Fortify or Breeze config.

---

## Severity guide

Use these defaults. Raise or lower a severity only with a stated reason.

| Severity | Items |
|---|---|
| Critical | Passwords not hashed; secrets exposed in client code or committed to git; storage buckets publicly listable; no idempotency on payment submissions; login or reset flow reveals whether an email exists AND has no rate limiting; open redirect after login; missing privacy policy or cookie consent where tracking scripts load |
| High | Missing security headers or HTTPS redirect; no server-side error monitoring; no connection pooling in serverless; unsafe account deletion; missing Terms of Service; analytics loading before consent; no backups; session expiry failing silently; password reset token not time-limited or single-use |
| Low | Missing `.env.example`; legal pages not linked in footer; cookie consent not persisted; no uptime monitor; weak but present CSP |