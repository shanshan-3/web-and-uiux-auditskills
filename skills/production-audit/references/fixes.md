# Fix Templates

Referenced by `production-audit` SKILL.md. Each section maps to a checklist item and provides a concrete fix.

## Contents

- Legal & Privacy
- Analytics & Monitoring
- Infrastructure & HTTPS
- Database & Backend Operations
- Auth & Critical Flows
- Laravel / PHP equivalents

---

## Legal & Privacy

**Missing privacy policy page**
Create a route with real content:
```
app/privacy/page.tsx    (Next.js App Router)
pages/privacy.tsx       (Next.js Pages Router)
```
Link from the footer:
```tsx
<Link href="/privacy">Privacy Policy</Link>
<Link href="/terms">Terms of Service</Link>
```
The policy must describe what data the site actually collects and which third parties receive it. Have it reviewed before launch if the site serves users in regulated regions.

**Missing cookie consent banner**
Using `react-cookie-consent`:
```bash
npm install react-cookie-consent
```
`react-cookie-consent` stores the choice in a cookie (default name `CookieConsent`), not in `localStorage`. Read it with `getCookieConsentValue()`. The banner's `onAccept` only fires when the visitor clicks, so returning visitors who already accepted need a check on mount:
```tsx
'use client';
import { useEffect } from 'react';
import CookieConsent, { getCookieConsentValue } from 'react-cookie-consent';

export default function ConsentBanner() {
  useEffect(() => {
    if (getCookieConsentValue() === 'true') loadAnalytics(); // returning visitor who accepted earlier
  }, []);

  return (
    <CookieConsent
      location="bottom"
      buttonText="Accept"
      declineButtonText="Decline"
      enableDeclineButton
      onAccept={() => loadAnalytics()}
    >
      This site uses cookies to improve your experience.
    </CookieConsent>
  );
}
```
`loadAnalytics` is your own function that injects the analytics script. It must not run before consent.

---

## Analytics & Monitoring

**No analytics configured**
Plausible does not use cookies, so it usually does not need a consent banner. Confirm this against the rules that apply to your audience.
```html
<script defer data-domain="example.com" src="https://plausible.io/js/script.js"></script>
```

Vercel Analytics (Next.js):
```bash
npm install @vercel/analytics
```
```tsx
import { Analytics } from '@vercel/analytics/react';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return <html><body>{children}<Analytics /></body></html>;
}
```
GA4 and PostHog (with cookies enabled) must load only after consent. Use the consent banner above.

**No error monitoring (Sentry)**
```bash
npx @sentry/wizard@latest -i nextjs
```
This creates the client and server config files and `instrumentation.ts`.

Add a global error boundary at `app/global-error.tsx`. Report the error in an effect, not during render, or it fires on every re-render:
```tsx
'use client';
import { useEffect } from 'react';
import * as Sentry from '@sentry/nextjs';

export default function GlobalError({ error }: { error: Error & { digest?: string } }) {
  useEffect(() => {
    Sentry.captureException(error);
  }, [error]);

  return (
    <html>
      <body>
        <h1>Something went wrong</h1>
      </body>
    </html>
  );
}
```

**No uptime monitoring - manual step**
1. Add a health route, for example `app/api/health/route.ts`:
```ts
export function GET() {
  return Response.json({ ok: true });
}
```
2. Create a monitor for the homepage and `/api/health` in UptimeRobot, BetterStack, or Checkly.
3. Send alerts to email or chat.

---

## Infrastructure & HTTPS

**HTTPS not enforced**
Vercel and Netlify redirect HTTP to HTTPS automatically; confirm it in the dashboard. For a self-hosted server:

Nginx:
```nginx
server {
  listen 80;
  server_name example.com www.example.com;
  return 301 https://$host$request_uri;
}
```
Express behind a proxy:
```js
app.set('trust proxy', 1);
app.use((req, res, next) => {
  if (req.secure) return next();
  res.redirect(301, `https://${req.headers.host}${req.url}`);
});
```
Replace any `http://` asset URLs in source. Find them while ignoring XML namespaces:
```bash
grep -rn "http://" --include="*.ts" --include="*.tsx" --include="*.css" --include="*.html" src/ | grep -v "w3.org"
```

**Missing security headers**
Next.js - add to `next.config.js`:
```js
module.exports = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
        ],
      },
    ];
  },
};
```
Only add `preload` to the HSTS header once every subdomain serves HTTPS, because preload is hard to undo.

Content-Security-Policy: start in report-only mode so you can see what would break before enforcing it:
```js
{ key: 'Content-Security-Policy-Report-Only', value: "default-src 'self'; img-src 'self' data: https:; style-src 'self' 'unsafe-inline'; script-src 'self'" }
```
Avoid `'unsafe-inline'` and `'unsafe-eval'` in `script-src` for the enforced policy. Next.js supports nonce-based CSP through middleware; see the Next.js "Content Security Policy" guide.

Vercel - add to `vercel.json`:
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" }
      ]
    }
  ]
}
```

**Exposed API keys in client code**
Scan the working tree, including `.env` files that should not be committed:
```bash
grep -rEn "sk_live|service_role|SECRET|PRIVATE" --include="*.ts" --include="*.tsx" --include="*.js" --include="*.jsx" src/
git ls-files | grep -E "(^|/)\.env($|\.)" | grep -v "\.example$"
```
Scan git history too, since a key removed from the code is still in older commits:
```bash
npx gitleaks detect
```
Move secrets server-side:
```ts
// BAD: exposed in the client bundle
const key = 'sk_live_abc123';

// GOOD: server-side only (not prefixed with NEXT_PUBLIC_ or VITE_)
const key = process.env.STRIPE_SECRET_KEY;
```
If a real secret was ever exposed, rotate it. Removing it from the code does not make it safe again.

**`.env` not ignored**
Add to `.gitignore`:
```
.env
.env.local
.env.*.local
```
If `.env` was already committed, untrack it with `git rm --cached .env` and rotate every secret in it.

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

**No database backups - manual step**
- Supabase: enable Point-in-Time Recovery or confirm daily backups in the project settings.
- Self-managed Postgres: schedule dumps, and store them off the server:
```bash
# crontab -e   (note the escaped % signs)
0 3 * * * pg_dump "$DATABASE_URL" -Fc -f /backups/db-$(date +\%F).dump
```
Test a restore at least once. A backup that has never been restored is not verified.

**No connection pooling (serverless)**
Supabase - use the pooled connection string, not the direct one. Copy the exact string from the dashboard (the Connect panel); the pooled host looks like `...pooler.supabase.com` and uses port 6543 in transaction mode:
```
# BAD: direct connection (exhausts connections in serverless)
DATABASE_URL=postgresql://postgres:[PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres

# GOOD: pooled connection via Supavisor (copy the real string from the dashboard)
DATABASE_URL=postgresql://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres?pgbouncer=true
DIRECT_URL=postgresql://postgres:[PASSWORD]@db.[PROJECT-REF].supabase.co:5432/postgres
```
Prisma - use the pooled URL for queries and the direct URL for migrations:
```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}
```

**No payment idempotency keys**
Generate the key once per checkout attempt and reuse it on every retry of that attempt. A key generated inside the request function changes on each retry and does not prevent duplicate charges:
```tsx
const idempotencyKey = useRef(crypto.randomUUID()); // one key per checkout attempt

async function pay() {
  const res = await fetch('/api/pay', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Idempotency-Key': idempotencyKey.current },
    body: JSON.stringify({ amount, currency }),
  });
  if (res.ok) idempotencyKey.current = crypto.randomUUID(); // new key for the next, separate purchase
}
```
Server side: enforce uniqueness in the database. A check-then-insert has a race condition when two requests arrive together.
```sql
ALTER TABLE payments ADD CONSTRAINT payments_idempotency_key_unique UNIQUE (idempotency_key);
```
```ts
try {
  const payment = await db.payment.create({ data: { idempotencyKey, amount, currency } });
  // charge the customer, then return the result
} catch (err: any) {
  if (err.code === 'P2002') { // Prisma unique constraint violation
    const existing = await db.payment.findUnique({ where: { idempotencyKey } });
    return Response.json(existing); // return the earlier result, do not charge again
  }
  throw err;
}
```
If you use Stripe, also pass the same key to Stripe: `stripe.paymentIntents.create({ ... }, { idempotencyKey })`.

**Unsafe account deletion**
Use soft-delete with a `deleted_at` timestamp:
```sql
ALTER TABLE users ADD COLUMN deleted_at timestamptz DEFAULT NULL;

-- "Delete" a user
UPDATE users SET deleted_at = now() WHERE id = $1;

-- All queries filter out deleted users
SELECT * FROM users WHERE deleted_at IS NULL;
```
Or define what happens to related rows with foreign keys:
```sql
ALTER TABLE orders ADD CONSTRAINT fk_user
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL;
```
`ON DELETE SET NULL` requires `orders.user_id` to be nullable. Use `ON DELETE CASCADE` only when the related rows should disappear with the user. If you must erase personal data, anonymize the soft-deleted row (blank the email and name) after your retention period.

**Storage buckets publicly listable**
Supabase - set the bucket to private and use signed URLs:
```ts
const { data } = await supabase.storage
  .from('private-uploads')
  .createSignedUrl('path/to/file.pdf', 3600); // 1 hour expiry
```
Add storage policies so users can only read and write their own folder.

---

## Auth & Critical Flows

**Weak password rules**
Prefer length over composition rules. Require a minimum of 12 characters and reject known-breached passwords:
```ts
const passwordSchema = z.string().min(12, 'At least 12 characters');
```
Breached-password check using the Have I Been Pwned range API. Only the first 5 characters of the SHA-1 hash leave your server:
```ts
import { createHash } from 'node:crypto';

export async function isBreached(password: string): Promise<boolean> {
  const sha1 = createHash('sha1').update(password).digest('hex').toUpperCase();
  const prefix = sha1.slice(0, 5);
  const suffix = sha1.slice(5);
  const res = await fetch(`https://api.pwnedpasswords.com/range/${prefix}`);
  const text = await res.text();
  return text.split('\n').some((line) => line.startsWith(suffix));
}
```

**Login reveals whether email exists**
```ts
// BAD: leaks user existence
if (!user) return res.status(404).json({ error: 'Email not found' });
if (!validPassword) return res.status(401).json({ error: 'Wrong password' });

// GOOD: neutral response for both cases
return res.status(401).json({ error: 'Invalid email or password' });
```
Also rate-limit the login route (see `website-audit` fixes, "No rate limiting on API routes") and compare passwords even when the user is not found, so response time does not reveal which emails exist.

**Password reset flow incomplete**
Store only a hash of the token, expire it, and make it single use:
```ts
import { randomBytes, createHash } from 'node:crypto';

const token = randomBytes(32).toString('hex');
const tokenHash = createHash('sha256').update(token).digest('hex');
await db.passwordReset.create({
  data: { userId: user.id, tokenHash, expiresAt: new Date(Date.now() + 60 * 60 * 1000) }, // 1 hour
});
// email a link containing `token`; always respond with the same neutral message

// On reset: look up by hash, reject if expired or already used, update the password, then delete the record
```

**No post-auth redirect to intended destination**
Save the intended URL before redirecting to login, and validate it afterwards. An unvalidated `redirect` parameter is an open redirect that attackers can use for phishing:
```ts
// Before redirect to login
const redirectTo = encodeURIComponent(window.location.pathname + window.location.search);
router.push(`/login?redirect=${redirectTo}`);

// After successful login
function safeRedirect(target: string | null, fallback = '/dashboard') {
  // Only same-site paths: must start with a single "/" and not "//" or "/\"
  if (target && target.startsWith('/') && !target.startsWith('//') && !target.startsWith('/\\')) {
    return target;
  }
  return fallback;
}
router.push(safeRedirect(searchParams.get('redirect')));
```

**Session expiry not handled**
Handle 401 responses in one place and send the user to login with a message:
```ts
async function api(input: RequestInfo, init?: RequestInit) {
  const res = await fetch(input, init);
  if (res.status === 401) {
    const redirectTo = encodeURIComponent(window.location.pathname);
    window.location.assign(`/login?expired=1&redirect=${redirectTo}`);
  }
  return res;
}
```
On the login page, show "Your session expired. Please sign in again." when `expired=1` is present.

**Passwords not hashed**
```ts
import bcrypt from 'bcrypt';

// On signup
const hashedPassword = await bcrypt.hash(password, 12);
await db.user.create({ data: { email, password: hashedPassword } });

// On login
const valid = await bcrypt.compare(password, user.password);
```
bcrypt only uses the first 72 bytes of a password. Argon2 (`argon2` package) is a good alternative with no such limit. Never hash with plain SHA-256 or MD5.

---

## Laravel / PHP equivalents

**Passwords hashed and validated**
```php
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;

$request->validate([
    'password' => ['required', Password::min(12)->uncompromised()],
]);

$user->password = Hash::make($request->password);
// Hash::check($plain, $user->password) on login
```

**Login throttling**
```php
Route::post('/login', [LoginController::class, 'store'])->middleware('throttle:5,1');
```

**Secrets, debug mode, and config**
- Keep `.env` in `.gitignore`. Set `APP_ENV=production` and `APP_DEBUG=false`.
- Read values with `config()` in application code; call `env()` only inside `config/*.php`.
- Run `php artisan config:cache` on deploy.

**HTTPS enforced**
```php
// app/Providers/AppServiceProvider.php, in boot()
if (app()->environment('production')) {
    \Illuminate\Support\Facades\URL::forceScheme('https');
}
```
Configure trusted proxies so Laravel detects HTTPS behind a load balancer.

**Security headers middleware**
```php
// php artisan make:middleware SecurityHeaders, then register it globally
public function handle($request, \Closure $next)
{
    $response = $next($request);
    $response->headers->set('X-Content-Type-Options', 'nosniff');
    $response->headers->set('X-Frame-Options', 'DENY');
    $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
    $response->headers->set('Strict-Transport-Security', 'max-age=63072000; includeSubDomains');
    return $response;
}
```

**Safe account deletion**
```php
// migration
$table->softDeletes();

// model
use Illuminate\Database\Eloquent\SoftDeletes;
class User extends Authenticatable { use SoftDeletes; }
```

**Private storage**
Use the `local` or a private S3 disk, and serve files through temporary URLs:
```php
$url = Storage::disk('s3')->temporaryUrl($path, now()->addHour());
```

**Backups**
Use `spatie/laravel-backup` and schedule `backup:run` and `backup:clean` in `routes/console.php` or `app/Console/Kernel.php`.