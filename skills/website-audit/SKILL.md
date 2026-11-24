---
name: website-audit
description: |
  Audits a website codebase and reports which standards are applied or missing, then suggests fixes for each gap. Triggers on: "audit the site", "audit this website", "check the website", "run audit", "what's missing". Use when reviewing a web project for responsiveness, performance, SEO, or validation completeness.
disable-model-invocation: true
license: MIT
---

Scan the project. For every item in each category below, determine its status and output a report. Read source files, config files, and public assets to verify — do not assume.

## Steps

1. Identify project root: locate `package.json`, `index.html`, or framework config (`next.config.*`, `vite.config.*`, `astro.config.*`).
2. Run each checklist category in order. For each item: read the relevant file(s), determine status, record result.
3. Output the full report as a markdown table per category (see format below).
4. After the table, list every ❌ and ⚠️ item with a concrete fix. See [fixes.md](references/fixes.md) for fix templates.
5. Done when every item has a status and every gap has a named fix.

## Report format

For each category, output:

```
### Category Name
| Item | Status | File / Evidence |
|---|---|---|
| Meta title tag | ✅ | src/app/layout.tsx:12 |
| robots.txt | ❌ | Not found in /public |
| .webp images | ⚠️ | 3 of 11 images still .png |
```

Status legend: ✅ Applied · ❌ Missing · ⚠️ Partial

---

## Checklist

### 1. Responsiveness
| Item | What to check |
|---|---|
| Desktop (1024–1400px) | Layout renders without overflow or collapsed columns |
| Tablet landscape (768–1024px) | Layout reflows correctly, no horizontal scroll |
| Large phone / small tablet (480–768px) | Columns stack, tap targets ≥44px |
| Mobile (320–480px) | No clipped text, no broken grids |
| Fluid scaling with `clamp()` | Font sizes and spacing use `clamp(min, preferred, max)` — not static `px` jumps at breakpoints |
| 3D canvas / heavy canvas hidden below 1100px | Canvas components wrapped in a width guard: hidden on screens narrower than 1100px |

Scan: CSS files, Tailwind classes, styled-components, or inline styles for breakpoints and `clamp()` usage.

### 2. Performance & Optimization
| Item | What to check |
|---|---|
| Images in .webp | All `/public` or `/assets` images are `.webp` (not `.jpg`/`.png`) |
| SVGs for icons | Icon imports use `.svg`, not `.png` icon sets |
| .woff2 fonts only | Font files or `@font-face` declarations use `.woff2` — not `.ttf` or `.otf` |
| Load only needed font weights | Only declared weights (e.g., 400, 600) are loaded — no wildcard `font-weight: 100 900` unless variable font |
| Lazy-loaded images | `<img loading="lazy">` or framework equivalent on below-fold images |
| Declared aspect ratios / fixed dimensions | Every `<img>` has explicit `width` + `height` or `aspect-ratio` to prevent layout shift (CLS) |
| Image compression (Cloudinary or equivalent) | Remote images served via CDN (Cloudinary, Imgix, Next/Image) with auto-compression |
| Image preloader / session storage preload | Critical above-fold images preloaded via `<link rel="preload">` or session-storage preload logic |
| Style preloading | Critical CSS loaded via `<link rel="preload" as="style">` or inlined in `<head>` |

Scan: `public/`, `assets/`, component imports, `<head>` tags, `next.config`, image component usage.

### 3. SEO
| Item | What to check |
|---|---|
| `<title>` tag | Present, unique per page, 50–60 chars |
| Meta description | Present, unique per page, 150–160 chars |
| Meta author | Present in `<head>` |
| `og:title` | Open Graph title set |
| `og:description` | Open Graph description set |
| `og:image` | Open Graph image set (1200×630px recommended) |
| `og:url` | Open Graph canonical URL set |
| `favicon.ico` | Present in `/public` or root |
| `favicon.png` | Present (32×32 or 64×64) |
| `robots.txt` | Present in `/public` or root; `Disallow` rules reviewed |
| `sitemap.xml` | Present or auto-generated; all public routes listed |
| Canonical tag | `<link rel="canonical">` on every page |
| Google Search Console | Note: cannot verify programmatically — flag for manual action if `robots.txt` or `sitemap.xml` is present but GSC submission unconfirmed |

Scan: `app/layout.*`, `pages/_document.*`, `<head>` components, `/public` directory, sitemap generator config.

### 4. Validation & Security
| Item | What to check |
|---|---|
| Broken links | Internal `<a href>` and `<Link>` targets resolve to real routes |
| Rate limiting | API routes have rate-limiting middleware (`express-rate-limit`, Upstash Ratelimit, or equivalent) |
| User input validation | All form submissions validated server-side (Zod, Yup, or equivalent schema) — not client-only |
| Button and field validation | Required fields enforced; submit disabled until valid |
| Error handling | API routes return structured error responses; unhandled rejections do not reach the client as raw stack traces |
| Row-level security (RLS) | If Supabase/Postgres: RLS enabled on all user-data tables; policies defined for select, insert, update, delete |

Scan: API route files, middleware config, DB migration files, Supabase dashboard config, form component files.

### 5. Animations & Motion
| Item | What to check |
|---|---|
| Scroll trigger animation | GSAP ScrollTrigger or Intersection Observer used; `will-change: transform` declared; animations cleaned up on unmount with `ctx.revert()` |
| Parallax effect | `transform: translateY()` used — not `top`/`left` (non-composited, causes layout thrash); disabled on mobile |
| Horizontal scroll animation | Overflow-x controlled; touch/swipe and keyboard navigation supported; no content clipping |
| Stacked animation | Z-index layers managed without conflicts; scroll-driven stacking uses `position: sticky` or ScrollTrigger pin |
| Lenis smooth scroll | Lenis installed and initialized; `lenis.raf()` hooked into animation frame; destroyed on unmount |
| GSAP animation | GSAP installed; ScrollTrigger plugin registered (`gsap.registerPlugin(ScrollTrigger)`); contexts cleaned up with `ctx.revert()` on unmount |
| Three.js animation | Resize handler attached and removed on unmount; `cancelAnimationFrame` called on cleanup; `devicePixelRatio` clamped to `Math.min(window.devicePixelRatio, 2)` |
| WebGL animation | Context loss handled (`webglcontextlost` event); graceful fallback for unsupported browsers; hidden or replaced on low-power / mobile devices |
| `prefers-reduced-motion` respected | All animations check `window.matchMedia('(prefers-reduced-motion: reduce)')` and disable or minimize motion accordingly |
| GPU-only properties animated | Animations use only `transform` and `opacity` — not `width`, `height`, `top`, `left`, `margin` (triggers layout) |
| Animation cleanup on unmount | Every animation instance, event listener, and RAF loop is destroyed when the component unmounts (no memory leaks) |

Scan: animation utility files, component files using GSAP/Three.js/WebGL/Lenis, `useEffect` cleanup returns, CSS animation declarations.
