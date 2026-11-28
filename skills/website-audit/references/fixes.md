# Fix Templates

Referenced by `website-audit` SKILL.md. Each section maps to a checklist item and provides a concrete fix.

Secrets in client code, HTTPS enforcement, and security headers are covered by `production-audit` (see its `references/fixes.md`).

## Contents

- Responsiveness
- Performance
- SEO
- Validation & Security
- Animations & Motion
- Laravel / PHP equivalents

---

## Responsiveness

**Missing `clamp()` scaling**
Replace static breakpoint font sizes with fluid values:
```css
font-size: clamp(1rem, 2.5vw, 1.5rem);
padding: clamp(1rem, 4vw, 2.5rem);
```
Use https://clamp.font-size.app to generate values from min/max px targets.

**3D canvas not hidden on small screens**
Hiding a canvas with CSS (`display: none`, `hidden min-[1100px]:block`) only hides it visually. The component still mounts, so WebGL still initializes and runs. Do not mount it at all below the breakpoint:
```tsx
'use client';
import dynamic from 'next/dynamic';
import { useEffect, useState } from 'react';

const HeavyCanvas = dynamic(() => import('./HeavyCanvas'), { ssr: false });

export function CanvasGuard() {
  const [wide, setWide] = useState(false);

  useEffect(() => {
    const mq = window.matchMedia('(min-width: 1100px)');
    const update = () => setWide(mq.matches);
    update();
    mq.addEventListener('change', update);
    return () => mq.removeEventListener('change', update);
  }, []);

  return wide ? <HeavyCanvas /> : null;
}
```
Outside React, use `matchMedia` the same way and only import and initialize the scene when it matches.

---

## Performance

**Images not .webp**
Convert with `cwebp` (install with `brew install webp` or `sudo apt install webp`):
```bash
find public -type f \( -iname '*.png' -o -iname '*.jpg' -o -iname '*.jpeg' \) -print0 |
  while IFS= read -r -d '' f; do cwebp -q 80 "$f" -o "${f%.*}.webp"; done
```
Or with `sharp` (Node 18.17+):
```js
// scripts/to-webp.mjs
import fs from 'node:fs';
import path from 'node:path';
import sharp from 'sharp';

for (const file of fs.readdirSync('public', { recursive: true })) {
  if (/\.(png|jpe?g)$/i.test(file)) {
    const src = path.join('public', file);
    await sharp(src).webp({ quality: 80 }).toFile(src.replace(/\.(png|jpe?g)$/i, '.webp'));
  }
}
```
Update all `<img src>` and CSS `url()` references, then delete the originals.

**No `loading="lazy"` on images**
Add to all below-fold images:
```html
<img src="..." alt="..." loading="lazy" width="800" height="600" />
```
In Next.js, `next/image` lazy-loads by default. Add `priority` only to above-fold hero images.

**Missing aspect ratio / dimensions**
Every `<img>` must declare width and height to prevent CLS:
```html
<img src="..." width="400" height="300" alt="..." />
```
Or with CSS: `aspect-ratio: 4 / 3;` on the container.

**Font not .woff2 or loading unused weights**
```css
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-400.woff2') format('woff2');
  font-weight: 400;
  font-display: swap;
}
```
Download only needed weights from https://fontsource.org or use a variable font with a declared range.

**No image CDN / compression**
Cloudinary auto-format and compress:
```
https://res.cloudinary.com/<cloud>/image/upload/f_auto,q_auto,w_800/<image>
```
In Next.js, configure `next.config.js`:
```js
images: { remotePatterns: [{ hostname: 'res.cloudinary.com' }] }
```

**Missing `<link rel="preload">` for images and fonts**
Add to `<head>` for above-fold images and critical fonts:
```html
<link rel="preload" as="image" href="/hero.webp" />
<link rel="preload" as="font" href="/fonts/inter-400.woff2" type="font/woff2" crossorigin />
```

**No style preloading**
Load critical CSS early without blocking render:
```html
<link rel="preload" as="style" href="/styles/critical.css" onload="this.onload=null;this.rel='stylesheet'" />
<noscript><link rel="stylesheet" href="/styles/critical.css" /></noscript>
```
Or inline the above-the-fold CSS in a `<style>` tag in `<head>`. Frameworks such as Next.js already inline and preload CSS; mark the item Applied when the framework handles it.

---

## SEO

**Missing meta tags**
In Next.js App Router (`layout.tsx`):
```ts
export const metadata: Metadata = {
  title: 'Page Title - Site Name',
  description: '150-160 char description.',
  authors: [{ name: 'Author Name' }],
  openGraph: {
    title: 'Page Title - Site Name',
    description: '150-160 char description.',
    url: 'https://example.com',
    images: [{ url: 'https://example.com/og.png', width: 1200, height: 630 }],
  },
};
```

**Missing favicon**
Place both in `/public`:
- `favicon.ico` (16x16 and 32x32 multi-size ICO)
- `favicon.png` (32x32 or 64x64 PNG)

Add to `<head>`:
```html
<link rel="icon" href="/favicon.ico" sizes="any" />
<link rel="icon" href="/favicon.png" type="image/png" />
```

**Missing robots.txt**
Create `/public/robots.txt`:
```
User-agent: *
Allow: /
Sitemap: https://example.com/sitemap.xml
```

**Missing sitemap.xml**
Next.js App Router - create `app/sitemap.ts`:
```ts
import type { MetadataRoute } from 'next';

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: 'https://example.com', lastModified: new Date() },
    { url: 'https://example.com/about', lastModified: new Date() },
  ];
}
```

**Missing canonical tag**
```html
<link rel="canonical" href="https://example.com/current-page" />
```
In Next.js metadata: `alternates: { canonical: 'https://example.com/current-page' }`.

**Google Search Console - manual step**
1. Go to https://search.google.com/search-console
2. Add property and enter the site URL
3. Verify via HTML tag or DNS record
4. Submit sitemap: Sitemaps, enter `sitemap.xml`, Submit

---

## Validation & Security

**No rate limiting on API routes**
Using Upstash Ratelimit (edge-compatible):
```ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),
});

export async function POST(req: Request) {
  const ip = req.headers.get('x-forwarded-for')?.split(',')[0]?.trim() ?? 'anonymous';
  const { success } = await ratelimit.limit(ip);
  if (!success) return new Response('Too many requests', { status: 429 });
  // ... handle the request
}
```
Apply a stricter limit to login, signup, password reset, and payment routes.

**Server-side validation missing**
Add Zod schema validation at the API boundary:
```ts
import { z } from 'zod';

const schema = z.object({ email: z.string().email(), name: z.string().min(1) });

export async function POST(req: Request) {
  const result = schema.safeParse(await req.json());
  if (!result.success) {
    return Response.json({ errors: result.error.flatten() }, { status: 400 });
  }
  // result.data is validated and typed
}
```

**Raw errors reach the client**
Catch errors, log them on the server only, and return a generic structured response:
```ts
export async function POST(req: Request) {
  try {
    // ... handle the request
    return Response.json({ ok: true });
  } catch (err) {
    console.error('POST /api/contact failed', err); // server log only
    return Response.json({ error: 'Something went wrong. Please try again.' }, { status: 500 });
  }
}
```
Never return `err.message` or `err.stack` to the client in production.

**RLS not enabled (Supabase)**
In Supabase dashboard, Table Editor, select the table, Enable RLS. Then add policies:
```sql
-- Users can only read their own rows
create policy "Users see own data" on profiles
  for select using (auth.uid() = user_id);
```
Verify: run `select * from pg_policies where tablename = 'profiles';` in the SQL editor.

**Broken links**
Run a link checker against the local dev server:
```bash
npx linkinator http://localhost:3000 --recurse --skip "^mailto:"
```
Fix 404 routes or update stale `href` values.

**No spam protection on public forms**
Add Cloudflare Turnstile (free):
```bash
npm install @marsidev/react-turnstile
```
```tsx
import { Turnstile } from '@marsidev/react-turnstile';

<Turnstile siteKey={process.env.NEXT_PUBLIC_TURNSTILE_SITE_KEY!} onSuccess={setToken} />
```
Verify server-side:
```ts
const res = await fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ secret: process.env.TURNSTILE_SECRET_KEY, response: token }),
});
const outcome = await res.json();
if (!outcome.success) return Response.json({ error: 'Verification failed' }, { status: 400 });
```
A honeypot field is a lightweight alternative: add a hidden input that real users never fill, and reject any submission where it has a value.

**Missing alt text on images**
Every `<img>` must have `alt`:
```html
<!-- Meaningful image -->
<img src="/team.webp" alt="Team photo at the 2026 company retreat" />

<!-- Decorative image -->
<img src="/divider.svg" alt="" role="presentation" />
```
In Next.js, `next/image` requires `alt` by default, and the lint rule `jsx-a11y/alt-text` catches missing ones.

---

## Animations & Motion

**Missing `prefers-reduced-motion` support**
With GSAP, use `matchMedia` so animations are created and reverted automatically:
```ts
const mm = gsap.matchMedia();
mm.add('(prefers-reduced-motion: no-preference)', () => {
  gsap.to('.hero', { y: -40, scrollTrigger: { trigger: '.hero', scrub: true } });
});
// on unmount: mm.revert();
```
Or check once:
```ts
const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
if (!prefersReduced) {
  gsap.to('.hero', { y: -40 });
}
```
CSS alternative:
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

**GSAP not cleaned up on unmount**
Always return a cleanup function from `useEffect`:
```ts
useEffect(() => {
  const ctx = gsap.context(() => {
    gsap.to('.box', { x: 100, scrollTrigger: { trigger: '.box', start: 'top center' } });
  }, containerRef);
  return () => ctx.revert(); // kills all animations and ScrollTriggers in this context
}, []);
```

**ScrollTrigger plugin not registered**
Register once at app root (not per component):
```ts
import { gsap } from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);
```

**Parallax using `top`/`left` (layout thrash)**
Replace with a GPU-composited transform:
```ts
// BAD: causes layout recalculation
element.style.top = `${offset}px`;

// GOOD: composited, no layout cost
element.style.transform = `translateY(${offset}px)`;
```
Disable parallax on mobile (adds jank on low-power devices):
```ts
if (window.matchMedia('(min-width: 769px)').matches) { initParallax(); }
```

**Horizontal scroll animation**
For a plain horizontal scroller, use native scrolling with snap, and make it keyboard reachable:
```html
<div class="h-scroll" role="region" aria-label="Projects" tabindex="0">
  <article>...</article>
  <article>...</article>
</div>
```
```css
.h-scroll {
  display: flex;
  gap: 1rem;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  overscroll-behavior-x: contain;
}
.h-scroll > * { flex: 0 0 min(80%, 24rem); scroll-snap-align: start; }
.h-scroll:focus-visible { outline: 2px solid currentColor; outline-offset: 2px; }
```
For scroll-pinned horizontal sections built with GSAP ScrollTrigger, keep the pinned section's content reachable by keyboard and revert the trigger on unmount.

**Stacked animation**
Use `position: sticky` with an incrementing offset and z-index instead of manual scroll math:
```html
<section class="stack">
  <article class="stack-card" style="--i: 1">...</article>
  <article class="stack-card" style="--i: 2">...</article>
  <article class="stack-card" style="--i: 3">...</article>
</section>
```
```css
.stack-card {
  position: sticky;
  top: calc(5rem + var(--i) * 1.5rem);
  z-index: var(--i);
}
```

**Lenis not destroyed on unmount**
```ts
useEffect(() => {
  const lenis = new Lenis({ duration: 1.2, easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)) });
  function raf(time: number) { lenis.raf(time); rafId = requestAnimationFrame(raf); }
  let rafId = requestAnimationFrame(raf);
  return () => { lenis.destroy(); cancelAnimationFrame(rafId); };
}, []);
```

**Three.js missing resize handler or cleanup**
```ts
useEffect(() => {
  // scene and camera are created earlier in this effect
  const renderer = new THREE.WebGLRenderer({ canvas: canvasRef.current! });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // clamp to 2

  function onResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
  window.addEventListener('resize', onResize);

  let rafId: number;
  function animate() { rafId = requestAnimationFrame(animate); renderer.render(scene, camera); }
  animate();

  return () => {
    window.removeEventListener('resize', onResize);
    cancelAnimationFrame(rafId);
    // dispose geometries, materials, and textures you created, then the renderer
    renderer.dispose();
  };
}, []);
```

**WebGL - no context loss handling**
```ts
canvas.addEventListener('webglcontextlost', (e) => {
  e.preventDefault();
  cancelAnimationFrame(rafId);
});
canvas.addEventListener('webglcontextrestored', () => {
  initWebGL(); // re-initialise shaders and buffers
});
```
Add a static fallback image inside the canvas container for unsupported browsers:
```html
<canvas id="gl-canvas">
  <img src="/fallback.webp" alt="Visual fallback" />
</canvas>
```

**Animating non-composited properties (causes layout)**
Only animate `transform` and `opacity`. Flag any animation of `width`, `height`, `top`, `left`, `margin`, `padding`:
```ts
// BAD: triggers layout on every frame
gsap.to(el, { width: 200 });

// GOOD: composited only
gsap.to(el, { scaleX: 1.5, opacity: 0 });
```
Use https://csstriggers.com to verify which properties trigger layout, paint, or composite only.

---

## Laravel / PHP equivalents

**Rate limiting**
```php
// routes/web.php or routes/api.php
Route::post('/contact', [ContactController::class, 'store'])->middleware('throttle:10,1');
```
Or define a named limiter:
```php
// app/Providers/AppServiceProvider.php, in boot()
RateLimiter::for('forms', fn (Request $request) => Limit::perMinute(10)->by($request->ip()));
```

**Server-side validation**
```php
// php artisan make:request ContactRequest
public function rules(): array
{
    return [
        'name'  => ['required', 'string', 'max:100'],
        'email' => ['required', 'email', 'max:255'],
    ];
}
```

**Raw errors reach the client**
Set `APP_DEBUG=false` and `APP_ENV=production` in the production `.env`. With debug on, Laravel returns stack traces to the browser.

**Spam protection (Turnstile)**
```php
$response = Http::asForm()->post('https://challenges.cloudflare.com/turnstile/v0/siteverify', [
    'secret'   => config('services.turnstile.secret'),
    'response' => $request->input('cf-turnstile-response'),
    'remoteip' => $request->ip(),
]);

if (! $response->json('success')) {
    abort(422, 'Verification failed');
}
```