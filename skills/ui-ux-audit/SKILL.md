---
name: ui-ux-audit
description: |
  Audits a website or app UI for interaction polish, form hardening, loading states, and visual feedback, then reports what is applied or missing. Triggers on: "audit UI", "review screen", "check button feedback", "check form validation", "check loading states", "check onboarding flow". Use when reviewing screens, forms, button states, loading behavior, or user-facing workflows.
disable-model-invocation: true
license: MIT
---

Scan component and page files. For every item in each category below, determine its status and output a report. Read source files — do not assume.

## Steps

1. Locate component files, page files, and form implementations in the project.
2. Run each checklist category. For each item: read the relevant file(s), determine status, record result.
3. Output the full report as a markdown table per category (see format below).
4. After every table, list each ❌ and ⚠️ item with a concrete fix.
5. Done when every item has a status and every gap has a named, actionable fix.

## Report Format

```
### Category Name
| Item | Status | File / Evidence |
|---|---|---|
| Submit button locks on first tap | ✅ | components/CheckoutForm.tsx:42 |
| Skeleton gated on isLoading only | ❌ | uses isFetching — skeletons stuck on refetch |
| Email lowercased on blur | ⚠️ | trimmed but not lowercased |
```

Status legend: ✅ Applied · ❌ Missing · ⚠️ Partial

---

## Checklist

### 1. Buttons & Touch Feedback
| Item | What to check |
|---|---|
| Submit/payment button locks on first tap | Button set to `disabled` immediately on click/tap while request is in-flight |
| Button dimensions frozen during loading | Width and height do not change when spinner replaces label text |
| Visual tap feedback within 100ms | Active state defined: `active:scale-[0.98]`, opacity shift, or background highlight |
| Action-oriented button labels | Labels use verb + noun (`Save Changes`, `Pay $49.00`) — not generic (`Submit`, `OK`, `Continue`) |

### 2. Loading States
| Item | What to check |
|---|---|
| Skeleton gated on `isLoading` only | Skeleton shows only when `data === undefined`; not gated on `isFetching` (causes stuck skeletons on refetch) |
| Stale-while-revalidate | Cached data renders immediately on revisit; refetch is silent in background |
| Skeleton dismissed as soon as `data` is defined | No delay waiting for network once a cache hit is available |
| Skeleton matches layout geometry | Skeleton placeholder reflects the actual content shape (card, list row, heading) with pulse animation |
| Spinners reserved for in-button micro-actions | Full-page or screen-level spinners replaced with skeletons |
| Splash screen held until auth resolves | Splash stays active until auth state and critical assets are ready (prevents FOUC / white flash) |
| Optimistic updates on mutations | Toggle, like, delete mutations update local state immediately; roll back with toast on failure |

### 3. Forms & Inputs
| Item | What to check |
|---|---|
| Email trimmed and lowercased on blur | `blur` handler trims whitespace and calls `.toLowerCase()` |
| Name fields accept apostrophes and Unicode | No restrictive `^[A-Za-z]+$` regex; allows `O'Connor`, `José`, `D'Angelo` |
| `autocomplete` attributes wired | `name`, `email`, `tel`, `current-password`, `new-password`, `address-line1`, `postal-code` all have explicit `autocomplete` |
| Validation fires on blur and submit only | No validation triggered on every keystroke |
| Inline error text anchored beneath field | Error messages positioned directly below the input with `aria-invalid="true"` and `aria-describedby` |
| Submit disabled until required fields valid | Primary action button remains disabled until basic validation passes |
| Date pickers have min/max constraints | Date inputs or pickers block impossible past or future dates before submission |

### 4. Media & Uploads
| Item | What to check |
|---|---|
| Client-side image compression before upload | Large images (mobile/Android camera) compressed in browser before network transfer |
| MIME type and file size validated client-side | Invalid file types rejected with error message; size limit stated in error text |
| Upload progress shown | Progress bar or percentage indicator shown during upload |

### 5. State & Resilience
| Item | What to check |
|---|---|
| Multi-step form state persisted | Draft state saved to `sessionStorage` or `localStorage` on input change; restored on reload |
| Timestamps stored as UTC, rendered as local time | All timestamps use ISO-8601/UTC internally; displayed in user's device timezone |
| Offline banner on connection drop | Non-intrusive banner appears when offline; in-flight state retained |
| "Tap to Retry" on reconnect | Explicit retry affordance shown when connection restores |

### 6. Onboarding, Help & Flows
| Item | What to check |
|---|---|
| Onboarding "Skip" button visible | Skip button present and prominent on all onboarding slides |
| Step indicator shown | `Step 2 of 4` or equivalent progress indicator displayed |
| FAQs use accessible accordion | `details`/`summary` or Radix/shadcn Accordion with keyboard support and open/close chevron |
| Password reset gives neutral feedback | Submission shows `"If an account exists for this email, a reset link has been sent."` — not `"Email not found"` |
| Resend button has cooldown | 60-second countdown enforced on resend; button disabled during cooldown |
| Zero console errors | No unhandled promise rejections, missing React keys, or hydration mismatches |
