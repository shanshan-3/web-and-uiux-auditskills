# Web & UI/UX Audit Skills

> AI coding agent skills that audit websites and web apps against fixed engineering checklists.

<p align="center">
  <img src="https://img.shields.io/badge/Format-Agent_Skills-8A2BE2?style=for-the-badge" alt="Agent Skills" />
  <img src="https://img.shields.io/badge/License-MIT-blue?style=for-the-badge" alt="License" />
  <img src="https://img.shields.io/badge/Works_With-Claude_Code-CC785C?style=for-the-badge&logo=anthropic&logoColor=white" alt="Claude Code" />
  <img src="https://img.shields.io/badge/Works_With-OpenCode-000000?style=for-the-badge" alt="OpenCode" />
  <img src="https://img.shields.io/badge/Works_With-Codex-412991?style=for-the-badge&logo=openai&logoColor=white" alt="Codex" />
</p>

---

## Overview

Each skill in this repository reads your codebase and reports what is applied and what is missing. Results are structured as evidence tables — every item gets a `✅ Applied`, `❌ Missing`, or `⚠️ Partial` status with a file path and line reference. Every gap gets a concrete fix.

Both skills are **user-invoked only** (`disable-model-invocation: true`). They run when you call them — not in the background on every file edit.

```mermaid
flowchart LR
    A["You call a skill"] --> B["Agent reads source files,\nconfig, and assets"]
    B --> C["Evidence table\nper checklist category"]
    C --> D["Named fix for\nevery gap"]
```

---

## Skills

### [`/ui-ux-audit`](skills/ui-ux-audit/SKILL.md)

Audits components and pages for interaction polish, form correctness, loading behavior, and user-flow completeness.

| Category | What Gets Checked |
|---|---|
| Buttons & Feedback | Submit locks on first tap, frozen button dimensions during loading, 100ms visual feedback, action-oriented labels, one clear CTA per screen, all interactive elements wired |
| Loading States | Skeleton gated on `isLoading` only (not `isFetching`), stale-while-revalidate, splash screen held until auth resolves |
| Forms & Inputs | Email trimmed and lowercased on `blur`, name fields allow Unicode and apostrophes, `autocomplete` wired, inline `aria-invalid` errors |
| Media & State | Client-side compression before upload, draft persistence to `sessionStorage`/`localStorage`, UTC storage with local timezone render |
| Flows & Help | FAQ accordions keyboard-accessible, onboarding has visible skip + step indicator, password reset gives neutral feedback with cooldown |
| Accessibility & Visual | WCAG AA color contrast (4.5:1 normal text, 3:1 large text), custom branded 404 page with recovery link |

---

### [`/website-audit`](skills/website-audit/SKILL.md)

Audits a full web project — located from `package.json`, `index.html`, or a framework config — across five technical categories.

| Category | What Gets Checked |
|---|---|
| Responsiveness | Layout across 320px–1400px, `clamp()` for fluid scaling, 3D canvas hidden below 1100px |
| Performance | `.webp` images, `.svg` icons, `.woff2` fonts, lazy loading, alt text on all images, layout shift prevention, Cloudinary or CDN delivery |
| SEO | `<title>`, `<meta description>`, Open Graph tags, `favicon.ico`/`.png`, canonical tags, `robots.txt`, `sitemap.xml` |
| Validation & Security | Broken links, rate-limited API routes, server-side Zod validation, Supabase RLS, exposed API keys scan, HTTPS enforcement, security headers, spam protection |
| Animations & Motion | GSAP `ctx.revert()` cleanup, Lenis lifecycle, Three.js resize and RAF disposal, WebGL context loss handling, GPU-only properties, `prefers-reduced-motion` |

Fix templates with copy-pasteable code for every failing item live in [`skills/website-audit/references/fixes.md`](skills/website-audit/references/fixes.md).

---

### [`/production-audit`](skills/production-audit/SKILL.md)

Audits a web project for production readiness — legal compliance, analytics, error monitoring, database operations, and auth flows.

| Category | What Gets Checked |
|---|---|
| Legal & Privacy | Privacy policy and Terms of Service pages exist and are linked in footer, cookie consent banner blocks non-essential scripts until accepted |
| Analytics & Monitoring | Analytics configured (GA4, Plausible, PostHog), client + server error monitoring (Sentry), uptime monitoring |
| Infrastructure & HTTPS | HTTPS enforced, security headers set, no exposed API keys in client code, `.env.example` present |
| Database & Backend Ops | Connection pooling for serverless, DB backups configured, payment idempotency keys, safe account deletion, private storage buckets |
| Auth & Critical Flows | Signup, login, and password reset flows complete with neutral error messages, post-auth redirect, session expiry handling, password hashing |

Fix templates live in [`skills/production-audit/references/fixes.md`](skills/production-audit/references/fixes.md).

---

## Installation

### Via `npx skills add` (Recommended)

Install directly to your current project:
```bash
npx skills add shanshan-3/web-and-uiux-auditskills
```

Target a specific agent using the `-a` flag:
```bash
# Claude Code
npx skills add shanshan-3/web-and-uiux-auditskills -a claude-code

# Cursor
npx skills add shanshan-3/web-and-uiux-auditskills -a cursor

# OpenCode
npx skills add shanshan-3/web-and-uiux-auditskills -a opencode
```

Install globally across all projects using `-g`:
```bash
npx skills add shanshan-3/web-and-uiux-auditskills -g
```

Or install an individual skill:
```bash
npx skills add shanshan-3/web-and-uiux-auditskills --skill ui-ux-audit
npx skills add shanshan-3/web-and-uiux-auditskills --skill website-audit
npx skills add shanshan-3/web-and-uiux-auditskills --skill production-audit
```

---

### Manual Installation by Agent

Clone the repository and copy the `skills/` folder to your agent's directory:

```bash
git clone https://github.com/shanshan-3/web-and-uiux-auditskills.git temp-skills
```

| Agent | Project Scope | Global Scope |
|---|---|---|
| **Claude Code** | `cp -r temp-skills/skills/* .claude/skills/` | `cp -r temp-skills/skills/* ~/.claude/skills/` |
| **Cursor** | `cp -r temp-skills/skills/* .cursor/skills/` | `cp -r temp-skills/skills/* ~/.agents/skills/` |
| **OpenCode** | `cp -r temp-skills/skills/* .opencode/skills/` | `cp -r temp-skills/skills/* ~/.config/opencode/skills/` |
| **Codex / Antigravity / Universal** | `cp -r temp-skills/skills/* .agents/skills/` | `cp -r temp-skills/skills/* ~/.agents/skills/` |

Clean up after copying:
```bash
rm -rf temp-skills
```

---

## Usage

In a Claude Code, OpenCode, Codex, Antigravity, or Cursor session:

```text
/ui-ux-audit
```
```text
/website-audit
```
```text
/production-audit
```

The agent reads your files, then outputs a table per category:

### Responsiveness
| Item | Status | File / Evidence |
|---|---|---|
| Fluid scaling with clamp() | ✅ | src/styles/typography.css:14 |
| Canvas hidden below 1100px | ❌ | components/HeroScene.tsx:28 — no viewport guard |
| Mobile layout (320px–480px) | ⚠️ | components/PricingTable.tsx:45 — overflow |

### Fixes
- HeroScene.tsx: wrap canvas in `hidden min-[1100px]:block`.
- PricingTable.tsx: stack grid to single column on mobile.

---

## Compatibility

All skills follow the [Agent Skills open standard](https://agentskills.io/specification). They work with any agent that supports `.agents/skills/` or `.claude/skills/` directories.

| Agent | Supported |
|---|---|
| Claude Code | ✅ |
| OpenCode | ✅ |
| Codex | ✅ |
| Google Antigravity | ✅ |
| Cursor | ✅ |

---

## License

[MIT](LICENSE)
