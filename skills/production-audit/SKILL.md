---
name: production-audit
description: |
  Audits a web project for production readiness: legal compliance, analytics, error monitoring, database operations, and infrastructure hardening. Triggers on: "production audit", "launch checklist", "ready to deploy", "pre-launch check", "go-live audit", "production readiness". Use when preparing to ship a website or web app to production.
disable-model-invocation: true
license: MIT
---

Scan the project for production readiness. For every item in each category below, determine its status and output a report. Read source files, config files, environment schemas, and public assets to verify — do not assume.

## Steps

1. Identify project root: locate `package.json`, framework config, and deployment config (`vercel.json`, `netlify.toml`, `Dockerfile`, etc.).
2. Run each checklist category. For each item: read the relevant file(s), determine status, record result.
3. Output the full report as a markdown table per category (see format below).
4. After every table, list each ❌ and ⚠️ item with a concrete fix. See [fixes.md](references/fixes.md) for fix templates.
5. Done when every item has a status and every gap has a named fix.

## Report Format

```
### Category Name
| Item | Status | File / Evidence |
|---|---|---|
| Privacy policy route | ✅ | app/privacy/page.tsx |
| Cookie consent banner | ❌ | No consent library or banner component found |
| Sentry SDK configured | ⚠️ | Client-side only — no server instrumentation |
```

Status legend: ✅ Applied · ❌ Missing · ⚠️ Partial

---

## Checklist

### 1. Legal & Privacy
| Item | What to check |
|---|---|
| Privacy policy page | Route exists (`/privacy` or `/privacy-policy`) with substantive content — not a placeholder |
| Terms of service page | Route exists (`/terms` or `/terms-of-service`) with substantive content |
| Footer links to legal pages | Privacy and Terms linked from the site footer or signup flow |
| Cookie consent banner | Consent banner present; non-essential tracking scripts (analytics, ads) blocked until user accepts |
| Cookie consent persisted | Consent choice saved to `localStorage` or cookie; banner does not reappear on every visit |

Scan: `app/` or `pages/` route files, footer/layout components, cookie consent library imports (`cookie-consent`, `react-cookie-consent`, `cookieyes`).

### 2. Analytics & Monitoring
| Item | What to check |
|---|---|
| Website analytics configured | Analytics script installed and initialized (GA4, Plausible, PostHog, Vercel Analytics, or Umami) |
| Analytics respects cookie consent | Analytics script loads only after user consent (if cookie banner is present) |
| Error monitoring (client) | Error boundary + monitoring SDK on the client (Sentry, LogRocket, Bugsnag, or equivalent) |
| Error monitoring (server) | Server-side error capture configured in API routes or middleware (Sentry `Sentry.init()` on server, or structured error logging) |
| Uptime monitoring | External uptime check configured (UptimeRobot, BetterStack, Checkly) — flag for manual action if not verifiable from code |

Scan: `app/layout.*`, `_app.*`, `instrumentation.ts`, `sentry.client.config.*`, `sentry.server.config.*`, analytics provider imports, middleware files.

### 3. Infrastructure & HTTPS
| Item | What to check |
|---|---|
| HTTPS enforced | Deployment config enforces HTTPS redirect; no mixed-content `http://` asset URLs in source |
| Security headers configured | `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy`, and `Content-Security-Policy` set via middleware, `next.config`, or hosting config |
| Environment variables secured | No API keys, secrets, or service-role keys hardcoded in client-side code; all secrets accessed via `process.env` server-side only; `.env` is in `.gitignore` |
| `.env.example` present | An `.env.example` or `.env.template` exists listing required variables without real values |

Scan: `next.config.*`, `vercel.json`, `netlify.toml`, middleware files, `headers()` config, `.gitignore`, `.env*` files, `grep` client bundles for `SECRET`, `PRIVATE`, `service_role`.

### 4. Database & Backend Operations
| Item | What to check |
|---|---|
| Database backups configured | Backup strategy documented or configured (Supabase PITR, managed Postgres backups, or cron-based `pg_dump`) — flag for manual verification if not in code |
| Connection pooling | Database connections use a pooler (Supabase Supavisor, PgBouncer, or Prisma connection pooling) — not direct connections in serverless |
| Safe account deletion | User deletion soft-deletes or cascades without breaking referential integrity; foreign key constraints or `ON DELETE` rules defined |
| Payment idempotency keys | Payment/transaction submissions include a client-generated idempotency key to prevent duplicate charges |
| Private storage buckets | File storage buckets (Supabase Storage, S3, R2) are not publicly listable; access controlled via signed URLs or RLS policies |

Scan: database migration files, schema files, Supabase config, storage bucket policies, payment API route files, user deletion handlers.

### 5. Auth & Critical Flows
| Item | What to check |
|---|---|
| Signup flow complete | Registration validates email format, enforces password strength, handles duplicate emails gracefully with a neutral message |
| Login flow complete | Login returns structured error on invalid credentials without revealing whether the email exists; rate-limited |
| Password reset flow complete | Reset sends email with time-limited token; reset page validates token expiry; success redirects to login |
| Post-auth redirect | After login/signup, user is redirected to their intended destination (not always the homepage) |
| Session expiry handled | Expired sessions redirect to login with a message; no silent failures or blank screens |
| Password hashing | Passwords hashed with bcrypt, scrypt, or Argon2 — never stored in plaintext or with MD5/SHA1 |

Scan: auth route files, middleware, session config, password handling in signup/login handlers, Supabase auth config.
