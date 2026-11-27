# Fix Templates

Referenced by `production-audit` SKILL.md. Each section maps to a checklist item and provides a concrete fix.

---

## Legal & Privacy

**Missing privacy policy page**
Create a route with real content:
```
app/privacy/page.tsx    (Next.js App Router)
pages/privacy.tsx       (Next.js Pages Router)
```
Link from footer:
```tsx
<Link href="/privacy">Privacy Policy</Link>
<Link href="/terms">Terms of Service</Link>
```

**Missing cookie consent banner**
Using `react-cookie-consent`:
```bash
npm install react-cookie-consent
```
```tsx
import CookieConsent from 'react-cookie-consent';

<CookieConsent
  location="bottom"
  buttonText="Accept"
  declineButtonText="Decline"
  enableDeclineButton
  onAccept={() => { loadAnalytics(); }}
>
  This site uses cookies to improve your experience.
</CookieConsent>
```
Block analytics until consent:
```ts
const consent = localStorage.getItem('CookieConsent');
if (consent === 'true') { loadAnalytics(); }
```

---

## Analytics & Monitoring

**No analytics configured**
Plausible (privacy-friendly, no cookie consent needed):
```html
<script defer data-domain="example.com" src="https://plausible.io/js/script.js"></script>
```

Vercel Analytics (Next.js):
```bash
npm install @vercel/analytics
```
```tsx
import { Analytics } from '@vercel/analytics/react';
export default function RootLayout({ children }) {
  return <html><body>{children}<Analytics /></body></html>;
}
```

**No error monitoring (Sentry)**
```bash
npx @sentry/wizard@latest -i nextjs
```
This creates `sentry.client.config.ts`, `sentry.server.config.ts`, and `instrumentation.ts`.

Add an error boundary in the root layout:
```tsx
'use client';
import * as Sentry from '@sentry/nextjs';

export default function GlobalError({ error }) {
  Sentry.captureException(error);
  return <html><body><h1>Something went wrong</h1></body></html>;
}
```

---

## Infrastructure & HTTPS

**Missing security headers**
Next.js — add to `next.config.js`:
```js
async headers() {
  return [{
    source: '/(.*)',
    headers: [
      { key: 'X-Content-Type-Options', value: 'nosniff' },
      { key: 'X-Frame-Options', value: 'DENY' },
      { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
      { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
      { key: 'Content-Security-Policy', value: "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" },
    ],
  }];
}
```

Vercel — add to `vercel.json`:
```json
{
  "headers": [
    { "source": "/(.*)", "headers": [
      { "key": "X-Content-Type-Options", "value": "nosniff" },
      { "key": "X-Frame-Options", "value": "DENY" }
    ]}
  ]
}
```

**Exposed API keys in client code**
Scan for leaks:
```bash
grep -r "sk_live\|service_role\|SECRET\|PRIVATE" --include="*.ts" --include="*.tsx" --include="*.js" src/
```
Move secrets server-side:
```ts
// ❌ Exposed in client bundle
const key = 'sk_live_abc123';

// ✅ Server-side only (not prefixed with NEXT_PUBLIC_)
const key = process.env.STRIPE_SECRET_KEY;
```

**Missing `.env.example`**
Create `.env.example` listing all required vars with blank values:
```
DATABASE_URL=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
SENTRY_DSN=
```

---

## Database & Backend Operations

**No connection pooling (serverless)**
Supabase — use the pooler URL (port 6543), not direct (port 5432):
```
# ❌ Direct connection (exhausts connections in serverless)
DATABASE_URL=postgresql://user:pass@db.xxx.supabase.co:5432/postgres

# ✅ Pooled connection via Supavisor
DATABASE_URL=postgresql://user:pass@db.xxx.supabase.co:6543/postgres?pgbouncer=true
```

Prisma — enable connection pooling:
```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}
```

**No payment idempotency keys**
Generate a client-side idempotency key per submission:
```ts
import { v4 as uuidv4 } from 'uuid';

const idempotencyKey = uuidv4();
await fetch('/api/pay', {
  method: 'POST',
  headers: { 'Idempotency-Key': idempotencyKey },
  body: JSON.stringify({ amount, currency }),
});
```
Server-side: check if the key was already processed before charging:
```ts
const existing = await db.payment.findUnique({ where: { idempotencyKey } });
if (existing) return Response.json(existing); // return cached result
```

**Unsafe account deletion**
Use soft-delete with a `deleted_at` timestamp:
```sql
ALTER TABLE users ADD COLUMN deleted_at timestamptz DEFAULT NULL;

-- "Delete" a user
UPDATE users SET deleted_at = now() WHERE id = $1;

-- All queries filter out deleted users
SELECT * FROM users WHERE deleted_at IS NULL;
```
Or cascade with foreign keys:
```sql
ALTER TABLE orders ADD CONSTRAINT fk_user
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL;
```

**Storage buckets publicly listable**
Supabase — set bucket to private and use signed URLs:
```ts
const { data } = await supabase.storage
  .from('private-uploads')
  .createSignedUrl('path/to/file.pdf', 3600); // 1 hour expiry
```

---

## Auth & Critical Flows

**No password strength enforcement**
```ts
const passwordSchema = z.string()
  .min(8, 'At least 8 characters')
  .regex(/[A-Z]/, 'One uppercase letter')
  .regex(/[0-9]/, 'One number')
  .regex(/[^A-Za-z0-9]/, 'One special character');
```

**Login reveals whether email exists**
```ts
// ❌ Leaks user existence
if (!user) return res.status(404).json({ error: 'Email not found' });
if (!validPassword) return res.status(401).json({ error: 'Wrong password' });

// ✅ Neutral response for both cases
return res.status(401).json({ error: 'Invalid email or password' });
```

**No post-auth redirect to intended destination**
Save the intended URL before redirecting to login:
```ts
// Before redirect to login
const redirectTo = encodeURIComponent(window.location.pathname);
router.push(`/login?redirect=${redirectTo}`);

// After successful login
const redirect = searchParams.get('redirect') || '/dashboard';
router.push(redirect);
```

**Passwords not hashed**
```ts
import bcrypt from 'bcrypt';

// On signup
const hashedPassword = await bcrypt.hash(password, 12);
await db.user.create({ data: { email, password: hashedPassword } });

// On login
const valid = await bcrypt.compare(password, user.password);
```
