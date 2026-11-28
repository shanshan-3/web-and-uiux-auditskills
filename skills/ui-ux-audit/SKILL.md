---
name: ui-ux-audit
description: |
  Audits a website or app UI for interaction polish, form hardening, loading states, accessibility basics, and visual feedback, then reports what is applied or missing. Triggers on: "audit UI", "review screen", "check button feedback", "check form validation", "check loading states", "check onboarding flow". Use when reviewing screens, forms, button states, loading behavior, or user-facing workflows.
disable-model-invocation: true
license: MIT
---

Scan component and page files. For every item in each category below, determine its status and output a report. Read source files. Do not assume.

## Steps

1. Identify the stack and locate component files, page files, and form implementations. Note which data-fetching library is used (React Query, SWR, none), which form library is used, and whether the project is React-based. Laravel projects: scan Blade views and Livewire or Vue components.
2. Run each checklist category. For each item: read the relevant file(s), determine status, record result.
3. Output the full report as a markdown table per category (see format below).
4. After every table, list each Missing and Partial item with a severity and a concrete fix. See [fixes.md](ui-ux-audit/references/fixes.md) for fix templates.
5. End the report with a "Manual checks" list covering every item marked Manual.
6. Done when every item has a status and every gap has a named, actionable fix.

## Rules

- Detect the stack first. If an item depends on a tool the project does not use (for example React Query or SWR for the loading-state rows, an upload feature, onboarding screens), mark it N/A and give a one-line reason. Never mark Missing for tooling or features the project does not have.
- If an item cannot be verified from source (runtime behavior, console output, visual judgment), mark it Manual and state exactly what the user should check. Never guess Applied.
- Give every Missing and Partial item a severity: Critical (security, data loss, duplicate payments), High (user-facing breakage, accessibility barriers), Low (polish). Sort the fix list by severity, Critical first.
- Cite file and line for every Applied, Missing, and Partial result.
- Do not use emojis in the report.

## Report Format

```
### Category Name
| Item | Status | File / Evidence |
|---|---|---|
| Submit button locks on first tap | Applied | components/CheckoutForm.tsx:42 |
| Skeleton gated on isLoading only | Missing | uses isFetching, so skeletons stick on refetch |
| Email lowercased on blur | Partial | trimmed but not lowercased |
| Optimistic updates on mutations | N/A | No data-fetching library or mutations in this project |
| Zero console errors | Manual | Run the app and check the browser console on each main page |
```

Then output the fix list:

```
### Fixes (sorted by severity)
1. [Critical] Submit button locks on first tap - components/CheckoutForm.tsx:42 allows double submit. Add a ref guard (see fixes.md, Buttons).
2. [Low] Email lowercased on blur - add .toLowerCase() to the blur handler.
```

Status legend: Applied, Missing, Partial, N/A, Manual.

---

## Checklist

### 1. Buttons & Touch Feedback
| Item | What to check |
|---|---|
| Submit/payment button locks on first tap | Button is disabled or guarded immediately on click/tap while the request is in flight, so a double tap cannot submit twice |
| Button dimensions frozen during loading | Width and height do not change when a spinner replaces the label text |
| Visual tap feedback | An active state is defined (`active:scale-[0.98]`, opacity shift, or background highlight). The 100ms response time itself cannot be measured from source; mark Manual if the feedback depends on JS state instead of CSS |
| Action-oriented button labels | Labels use verb + noun (`Save Changes`, `Pay $49.00`), not generic (`Submit`, `OK`, `Continue`) |
| All interactive elements wired | No dead `onClick={() => {}}` handlers, no `href="#"` placeholder links, no buttons without event handlers |
| One clear primary CTA per screen | Each screen or hero section has a single visually dominant primary action. This needs visual judgment: report what the markup suggests and mark Manual |

### 2. Loading States
| Item | What to check |
|---|---|
| Skeleton gated on `isLoading` only | Skeleton shows only when `data === undefined`; not gated on `isFetching` (causes stuck skeletons on refetch). N/A without React Query, SWR, or similar |
| Stale-while-revalidate | Cached data renders immediately on revisit; refetch is silent in the background. N/A without a caching data layer |
| Skeleton dismissed as soon as `data` is defined | No delay waiting for the network once a cache hit is available |
| Skeleton matches layout geometry | Skeleton placeholder reflects the actual content shape (card, list row, heading) with a pulse animation |
| Spinners reserved for in-button micro-actions | Full-page or screen-level spinners replaced with skeletons |
| Splash screen held until auth resolves | Splash stays active until auth state and critical assets are ready (prevents FOUC and white flash). N/A if the app has no auth or splash |
| Optimistic updates on mutations | Toggle, like, and delete mutations update local state immediately and roll back with a toast on failure. N/A if there are no such mutations |

### 3. Forms & Inputs
| Item | What to check |
|---|---|
| Email trimmed and lowercased on blur | `blur` handler trims whitespace and calls `.toLowerCase()` |
| Name fields accept apostrophes and Unicode | No restrictive `^[A-Za-z]+$` regex; allows `O'Connor`, `Jose`, `D'Angelo`, and non-Latin names |
| `autocomplete` attributes wired | `name`, `email`, `tel`, `current-password`, `new-password`, `address-line1`, `postal-code` all have explicit `autocomplete` |
| Validation fires on blur and submit only | No validation triggered on every keystroke before the field has been touched |
| Inline error text anchored beneath field | Error messages positioned directly below the input with `aria-invalid="true"` and `aria-describedby` |
| Invalid submit explains itself | On submit with invalid fields, the form shows inline errors and moves focus to the first invalid field. A submit button that is disabled for the whole time the form is invalid, with no message saying why, is Partial. Disabling during an in-flight request is fine |
| Date pickers have min/max constraints | Date inputs or pickers block impossible past or future dates before submission |

### 4. Media & Uploads
| Item | What to check |
|---|---|
| Client-side image compression before upload | Large images (mobile and Android camera) compressed in the browser before network transfer. N/A if the project has no uploads |
| MIME type and file size validated client-side | Invalid file types rejected with an error message; size limit stated in the error text |
| Upload progress shown | Progress bar or percentage indicator shown during upload |

### 5. State & Resilience
| Item | What to check |
|---|---|
| Multi-step form state persisted | Draft state saved to `sessionStorage` or `localStorage` on input change and restored on reload. Passwords and card numbers must not be persisted. N/A if there are no multi-step forms |
| Timestamps stored as UTC, rendered as local time | All timestamps use ISO-8601/UTC internally; displayed in the user's device timezone |
| Offline banner on connection drop | Non-intrusive banner appears when offline; in-flight state retained |
| "Tap to Retry" on reconnect | Explicit retry affordance shown when connection restores |

### 6. Onboarding, Help & Flows
| Item | What to check |
|---|---|
| Onboarding "Skip" button visible | Skip button present and prominent on all onboarding slides. N/A if there is no onboarding |
| Step indicator shown | `Step 2 of 4` or equivalent progress indicator displayed. N/A if there is no multi-step flow |
| FAQs use accessible accordion | `details`/`summary` or Radix/shadcn Accordion with keyboard support and an open/close chevron. N/A if there is no FAQ |
| Password reset gives neutral feedback | Submission shows `"If an account exists for this email, a reset link has been sent."`, not `"Email not found"`. N/A if there is no password reset |
| Resend button has cooldown | 60-second countdown enforced on resend; button disabled during cooldown |
| Zero console errors | No unhandled promise rejections, missing React keys, or hydration mismatches. Needs a running app, so mark Manual; static evidence (lint output for `react/jsx-key`) can support a Partial |

### 7. Accessibility & Visual
| Item | What to check |
|---|---|
| Color contrast meets WCAG AA | Normal text at least 4.5:1 contrast ratio; large text (18px+ or 14px bold) and UI components at least 3:1. Compute from the color tokens in source where possible; otherwise Manual |
| Visible focus styles | Interactive elements show a visible focus indicator (`:focus-visible` styles or framework defaults). Flag any `outline: none` or `outline: 0` that has no replacement |
| Form labels associated | Every input has a `<label>` linked by `for`/`htmlFor` or wrapping, or an `aria-label`. A placeholder alone is not a label |
| Custom 404 page | A branded `not-found` / `404` page exists with a clear link back to the homepage or dashboard, not the default framework 404 |

---

## Severity guide

Use these defaults. Raise or lower a severity only with a stated reason.

| Severity | Items |
|---|---|
| Critical | Submit or payment button can be double-submitted |
| High | Dead handlers or placeholder links; skeletons stuck on refetch; password reset reveals whether an email exists; inputs without labels; removed focus outlines with no replacement; color contrast failures; name fields rejecting valid names; invalid submit with no explanation |
| Low | Generic button labels; missing tap feedback; frozen dimensions; missing autocomplete; email not lowercased; missing step indicator or skip button; missing offline banner; missing custom 404; missing resend cooldown |