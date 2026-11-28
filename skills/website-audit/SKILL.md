---
name: website-audit
description: |
  Audits a website codebase and reports which standards are applied or missing, then suggests fixes for each gap. Covers responsiveness, performance, SEO, input validation, and animation quality. Triggers on: "audit the site", "audit this website", "check the website", "run audit", "what's missing". Use when reviewing a web project for responsiveness, performance, SEO, validation, or animation completeness. Secrets, HTTPS, and security headers are covered by production-audit.
disable-model-invocation: true
license: MIT
---

Scan the project. For every item in each category below, determine its status and output a report. Read source files, config files, and public assets to verify. Do not assume.

## Steps

1. Identify the project root and the stack. Locate `package.json`, `composer.json` or `artisan` (Laravel), `index.html`, or a framework config (`next.config.*`, `vite.config.*`, `astro.config.*`). Note which of these are present: animation libraries (GSAP, Lenis, Three.js), a database or backend service (Supabase, Prisma, Eloquent), and the styling approach (Tailwind, plain CSS, styled-components).
2. Run each checklist category in order. For each item: read the relevant file(s), determine status, record result.
3. Output the full report as a markdown table per category (see format below).
4. After each table, list every Missing and Partial item with a severity and a concrete fix. See [fixes.md](references/fixes.md) for fix templates.
5. End the report with a "Manual checks" list covering every item marked Manual.
6. Done when every item has a status and every gap has a named fix.

## Rules

- Detect the stack first. If an item depends on a tool the project does not use (for example GSAP, Three.js, Supabase, Cloudinary), mark it N/A and give a one-line reason. Never mark Missing for tooling the project does not need.
- If an item cannot be verified from source (rendering at a specific viewport, dashboard settings, anything that needs a running browser), mark it Manual and state exactly what the user should check. Never guess Applied.
- Give every Missing and Partial item a severity: Critical (security, data loss, legal exposure), High (user-facing breakage, SEO or performance blockers), Low (polish). Sort the fix list by severity, Critical first.
- Cite file and line for every Applied, Missing, and Partial result.
- Do not use emojis in the report.

## Report format

For each category, output:

```
### Category Name
| Item | Status | File / Evidence |
|---|---|---|
| Meta title tag | Applied | src/app/layout.tsx:12 |
| robots.txt | Missing | Not found in /public |
| .webp images | Partial | 3 of 11 images still .png |
| Lenis smooth scroll | N/A | Lenis is not installed |
| Google Search Console | Manual | Confirm the sitemap is submitted |
```

Then output the fix list:

```
### Fixes (sorted by severity)
1. [High] robots.txt - create public/robots.txt (see fixes.md, SEO).
2. [Low] .webp images - convert the 3 remaining .png files in public/images.
```

Status legend: Applied, Missing, Partial, N/A, Manual.

---

## Checklist

### 1. Responsiveness
| Item | What to check |
|---|---|
| Desktop (1024-1400px) | Layout renders without overflow or collapsed columns |
| Tablet landscape (768-1024px) | Layout reflows correctly, no horizontal scroll |
| Large phone / small tablet (480-768px) | Columns stack, tap targets at least 44px |
| Mobile (320-480px) | No clipped text, no broken grids |
| Fluid scaling with `clamp()` | Font sizes and spacing use `clamp(min, preferred, max)`, not static `px` jumps at breakpoints |
| 3D canvas / heavy canvas not mounted below 1100px | Canvas components are conditionally rendered (not just hidden with CSS) on screens narrower than 1100px |

The four viewport rows cannot be fully confirmed from source. Report what the breakpoints, grid and flex rules, and overflow settings show, then mark the row Manual and name the widths to test in browser devtools.

Scan: CSS files, Tailwind classes, styled-components, or inline styles for breakpoints and `clamp()` usage.

### 2. Performance & Optimization
| Item | What to check |
|---|---|
| Images in .webp | All `/public` or `/assets` images are `.webp` (not `.jpg`/`.png`) |
| SVGs for icons | Icon imports use `.svg`, not `.png` icon sets |
| .woff2 fonts only | Font files or `@font-face` declarations use `.woff2`, not `.ttf` or `.otf` |
| Load only needed font weights | Only declared weights (for example 400, 600) are loaded; no wildcard `font-weight: 100 900` unless it is a variable font |
| Lazy-loaded images | `<img loading="lazy">` or framework equivalent on below-fold images |
| Declared aspect ratios / fixed dimensions | Every `<img>` has explicit `width` and `height` or `aspect-ratio` to prevent layout shift (CLS) |
| Alt text on all images | Every `<img>` has descriptive `alt` text; decorative images use `alt=""` explicitly |
| Image compression (Cloudinary or equivalent) | Remote images served via a CDN (Cloudinary, Imgix, Next/Image) with auto-compression. Mark N/A if the project has no remote images |
| Image preloader | Critical above-fold images preloaded via `<link rel="preload">` or equivalent |
| Style preloading | Critical CSS loaded via `<link rel="preload" as="style">` or inlined in `<head>` |

Scan: `public/`, `assets/`, component imports, `<head>` tags, `next.config`, image component usage.

### 3. SEO
| Item | What to check |
|---|---|
| `<title>` tag | Present, unique per page, 50-60 chars |
| Meta description | Present, unique per page, 150-160 chars |
| Meta author | Present in `<head>` |
| `og:title` | Open Graph title set |
| `og:description` | Open Graph description set |
| `og:image` | Open Graph image set (1200x630px recommended) |
| `og:url` | Open Graph canonical URL set |
| `favicon.ico` | Present in `/public` or root |
| `favicon.png` | Present (32x32 or 64x64) |
| `robots.txt` | Present in `/public` or root; `Disallow` rules reviewed |
| `sitemap.xml` | Present or auto-generated; all public routes listed |
| Canonical tag | `<link rel="canonical">` on every page |
| Google Search Console | Cannot be verified from source. Always mark Manual, and ask the user to confirm the sitemap is submitted |

Scan: `app/layout.*`, `pages/_document.*`, `<head>` components, `/public` directory, sitemap generator config.

### 4. Validation & Security
| Item | What to check |
|---|---|
| Broken links | Internal `<a href>` and `<Link>` targets resolve to real routes |
| Rate limiting | API routes have rate-limiting middleware (`express-rate-limit`, Upstash Ratelimit, Laravel `throttle`, or equivalent). Mark N/A if there are no API routes |
| User input validation | All form submissions validated server-side (Zod, Yup, Laravel `FormRequest`, or equivalent schema), not client-only |
| Error handling | API routes return structured error responses; unhandled errors do not reach the client as raw stack traces |
| Row-level security (RLS) | If Supabase/Postgres: RLS enabled on all user-data tables; policies defined for select, insert, update, delete. Mark N/A otherwise |
| Spam protection on public forms | Contact, signup, or public-facing forms protected by Turnstile, hCaptcha, reCAPTCHA, or a honeypot field |

Secrets in client code, HTTPS enforcement, and security headers are checked by `/production-audit` (section 3). Do not repeat those checks here. Client-side button and field validation is checked by `/ui-ux-audit`.

Scan: API route files, middleware config, DB migration files, Supabase config, form component files.

### 5. Animations & Motion
| Item | What to check |
|---|---|
| Scroll trigger animation | GSAP ScrollTrigger or Intersection Observer used; `will-change: transform` declared; animations cleaned up on unmount with `ctx.revert()` |
| Parallax effect | `transform: translateY()` used, not `top`/`left` (non-composited, causes layout thrash); disabled on mobile |
| Horizontal scroll animation | Overflow-x controlled; touch/swipe and keyboard navigation supported; no content clipping |
| Stacked animation | Z-index layers managed without conflicts; scroll-driven stacking uses `position: sticky` or ScrollTrigger pin |
| Lenis smooth scroll | Lenis installed and initialized; `lenis.raf()` hooked into animation frame; destroyed on unmount |
| GSAP animation | GSAP installed; ScrollTrigger plugin registered (`gsap.registerPlugin(ScrollTrigger)`); contexts cleaned up with `ctx.revert()` on unmount |
| Three.js animation | Resize handler attached and removed on unmount; `cancelAnimationFrame` called on cleanup; `devicePixelRatio` clamped to `Math.min(window.devicePixelRatio, 2)` |
| WebGL animation | Context loss handled (`webglcontextlost` event); graceful fallback for unsupported browsers; hidden or replaced on low-power or mobile devices |
| `prefers-reduced-motion` respected | All animations check `window.matchMedia('(prefers-reduced-motion: reduce)')` and disable or minimize motion accordingly |
| GPU-only properties animated | Animations use only `transform` and `opacity`, not `width`, `height`, `top`, `left`, `margin` (triggers layout) |
| Animation cleanup on unmount | Every animation instance, event listener, and RAF loop is destroyed when the component unmounts (no memory leaks) |

Mark every animation row N/A when the project has no animation library and no custom animation code. Do not mark `prefers-reduced-motion` N/A if any CSS animation or transition exists.

Scan: animation utility files, component files using GSAP/Three.js/WebGL/Lenis, `useEffect` cleanup returns, CSS animation declarations.

---

## Severity guide

Use these defaults. Raise or lower a severity only with a stated reason.

| Severity | Items |
|---|---|
| Critical | RLS missing on tables holding user data; no server-side validation on endpoints that write data; raw stack traces returned to clients |
| High | No rate limiting on public or auth endpoints; no spam protection on public forms; missing title or meta description; missing `robots.txt`, `sitemap.xml`, or canonical tags; images without `width`/`height` or alt text; broken internal links; animation leaks (no cleanup on unmount); missing `prefers-reduced-motion`; layout breakage at a tested viewport |
| Low | Missing meta author, `favicon.png`, or `og:url`; images not `.webp`; fonts not `.woff2` or extra weights; missing preloads; SVG icon format; non-composited animation properties on small elements |