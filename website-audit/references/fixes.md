# Fix Templates

Referenced by `website-audit` SKILL.md. Each section maps to a checklist item and provides a concrete fix.

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
Wrap canvas in a responsive guard:
```tsx
// Tailwind
<div className="hidden min-[1100px]:block">
  <HeavyCanvas />
</div>

// CSS
@media (max-width: 1099px) { .canvas-wrapper { display: none; } }
```

---

## Performance

**Images not .webp**
Batch convert with `sharp` or `cwebp`:
```bash
npx @squoosh/cli --webp '{}' public/images/*.jpg
# or
find public -name "*.png" -exec cwebp {} -o {}.webp \;
```
Update all `<img src>` and CSS `url()` references.

**No `loading="lazy"` on images**
Add to all below-fold images:
```html
<img src="..." alt="..." loading="lazy" width="800" height="600" />
```
In Next.js, `next/image` lazy-loads by default. Add `priority` only to above-fold hero images.

**Missing aspect ratio / dimensions**
Every `<img>` must declare width + height to prevent CLS:
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
Cloudinary auto-format + compress:
```
https://res.cloudinary.com/<cloud>/image/upload/f_auto,q_auto,w_800/<image>
```
In Next.js, configure `next.config.js`:
```js
images: { remotePatterns: [{ hostname: 'res.cloudinary.com' }] }
```

**Missing `<link rel="preload">`**
Add to `<head>` for above-fold images and critical fonts:
```html
<link rel="preload" as="image" href="/hero.webp" />
<link rel="preload" as="font" href="/fonts/inter-400.woff2" type="font/woff2" crossorigin />
```

---

## SEO

**Missing meta tags**
In Next.js App Router (`layout.tsx`):
```ts
export const metadata: Metadata = {
  title: 'Page Title — Site Name',
  description: '150–160 char description.',
  authors: [{ name: 'Author Name' }],
  openGraph: {
    title: 'Page Title — Site Name',
    description: '150–160 char description.',
    url: 'https://example.com',
    images: [{ url: 'https://example.com/og.png', width: 1200, height: 630 }],
  },
};
```

**Missing favicon**
Place both in `/public`:
- `favicon.ico` (16×16, 32×32 multi-size ICO)
- `favicon.png` (32×32 or 64×64 PNG)

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
Next.js App Router — create `app/sitemap.ts`:
```ts
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

**Google Search Console — manual step**
1. Go to https://search.google.com/search-console
2. Add property → enter site URL
3. Verify via HTML tag or DNS record
4. Submit sitemap: Sitemaps → enter `sitemap.xml` → Submit

---

## Validation & Security

**No rate limiting on API routes**
Using Upstash Ratelimit (edge-compatible):
```ts
import { Ratelimit } from '@upstash/ratelimit';
const ratelimit = new Ratelimit({ redis, limiter: Ratelimit.slidingWindow(10, '10 s') });

const { success } = await ratelimit.limit(ip);
if (!success) return new Response('Too many requests', { status: 429 });
```

**Server-side validation missing**
Add Zod schema validation at the API boundary:
```ts
const schema = z.object({ email: z.string().email(), name: z.string().min(1) });
const result = schema.safeParse(req.body);
if (!result.success) return res.status(400).json({ errors: result.error.flatten() });
```

**RLS not enabled (Supabase)**
In Supabase dashboard → Table Editor → select table → Enable RLS.
Then add policies:
```sql
-- Users can only read their own rows
create policy "Users see own data" on profiles
  for select using (auth.uid() = user_id);
```
Verify: run `select * from pg_policies where tablename = 'profiles';` in SQL editor.

**Broken links**
Run a link checker:
```bash
npx linkinator https://localhost:3000 --recurse --skip "^mailto"
```
Fix 404 routes or update stale `href` values.

---

## Animations & Motion

**Missing `prefers-reduced-motion` support**
Wrap all GSAP / CSS animations in a motion check:
```ts
const prefersReduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
if (!prefersReduced) {
  gsap.to('.hero', { y: -40, scrollTrigger: { ... } });
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
  return () => ctx.revert(); // kills all animations and ScrollTriggers in context
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
Replace with GPU-composited transform:
```ts
// ❌ causes layout recalculation
element.style.top = `${offset}px`;

// ✅ composited, no layout cost
element.style.transform = `translateY(${offset}px)`;
```
Disable parallax on mobile (adds jank on low-power devices):
```ts
if (window.innerWidth > 768) { initParallax(); }
```

**Lenis not destroyed on unmount**
```ts
useEffect(() => {
  const lenis = new Lenis({ duration: 1.2, easing: (t) => Math.min(1, 1.001 - Math.pow(2, -10 * t)) });
  function raf(time: number) { lenis.raf(time); requestAnimationFrame(raf); }
  const rafId = requestAnimationFrame(raf);
  return () => { lenis.destroy(); cancelAnimationFrame(rafId); };
}, []);
```

**Three.js missing resize handler or cleanup**
```ts
useEffect(() => {
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
    renderer.dispose();
  };
}, []);
```

**WebGL — no context loss handling**
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
// ❌ triggers layout on every frame
gsap.to(el, { width: 200 });

// ✅ composited only
gsap.to(el, { scaleX: 1.5, opacity: 0 });
```
Use https://csstriggers.com to verify which properties trigger layout, paint, or composite only.
