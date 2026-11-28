# Fix Templates

Referenced by `ui-ux-audit` SKILL.md. Each section maps to a checklist item and provides a concrete fix. Examples use React; adapt the idea to Vue, Svelte, or Blade/Livewire.

## Contents

- Buttons & Touch Feedback
- Loading States
- Forms & Inputs
- Media & Uploads
- State & Resilience
- Onboarding, Help & Flows
- Accessibility & Visual

---

## Buttons & Touch Feedback

**Submit button does not lock on first tap**
State updates are asynchronous, so two fast taps can both pass an `isSubmitting` check. Guard with a ref as well:
```tsx
const submitting = useRef(false);
const [isSubmitting, setIsSubmitting] = useState(false);

async function onSubmit(e: React.FormEvent) {
  e.preventDefault();
  if (submitting.current) return;
  submitting.current = true;
  setIsSubmitting(true);
  try {
    await pay();
  } finally {
    submitting.current = false;
    setIsSubmitting(false);
  }
}
```
For payments, also send an idempotency key (see `production-audit` fixes, "No payment idempotency keys").

**Button dimensions change during loading**
Keep the label in the layout and overlay the spinner:
```tsx
<button type="submit" disabled={isSubmitting} aria-busy={isSubmitting} className="relative">
  <span className={isSubmitting ? 'invisible' : ''}>Pay $49.00</span>
  {isSubmitting && <Spinner className="absolute inset-0 m-auto" />}
</button>
```

**No visual tap feedback**
```css
.btn { transition: transform 80ms ease, opacity 80ms ease; }
.btn:active { transform: scale(0.98); opacity: 0.9; }
```
Tailwind: `active:scale-[0.98] active:opacity-90 transition`.

**Generic button labels**
Use verb + noun: `Save Changes`, `Send Message`, `Create Account`, `Pay $49.00`. Avoid `Submit`, `OK`, and `Continue` unless the next step is obvious from context.

**Dead handlers and placeholder links**
Find them:
```bash
grep -rnE "onClick=\{\(\) => \{\}\}|href=\"#\"" src/
```
Wire each one to a real action or route, or remove the element.

---

## Loading States

**Skeleton gated on `isFetching`**
Show the skeleton only while there is no data. Use `isFetching` for a subtle background indicator:
```tsx
const { data, isLoading, isFetching } = useQuery({ queryKey: ['posts'], queryFn: getPosts });

if (isLoading) return <PostListSkeleton />;   // first load, no cached data
return (
  <>
    {isFetching && <TopBarSpinner />}          // silent background refresh
    <PostList posts={data} />
  </>
);
```

**Stale-while-revalidate**
Give cached data a stale time so revisits render instantly:
```ts
useQuery({ queryKey: ['posts'], queryFn: getPosts, staleTime: 60_000 });
```

**Skeleton does not match layout**
Reuse the real component's dimensions:
```tsx
function CardSkeleton() {
  return <div className="h-40 w-full animate-pulse rounded-lg bg-gray-200" aria-hidden="true" />;
}
```

**Full-page spinners**
Replace screen-level spinners with skeletons. Keep spinners inside buttons only.

**Splash screen shows white flash**
Hold the splash until auth resolves:
```tsx
const { status } = useAuth(); // 'loading' | 'authed' | 'guest'
if (status === 'loading') return <Splash />;
```

**No optimistic updates**
```ts
useMutation({
  mutationFn: toggleLike,
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ['posts'] });
    const previous = queryClient.getQueryData(['posts']);
    queryClient.setQueryData(['posts'], (old: Post[]) =>
      old.map((p) => (p.id === id ? { ...p, liked: !p.liked } : p)));
    return { previous };
  },
  onError: (_err, _id, ctx) => {
    queryClient.setQueryData(['posts'], ctx?.previous);
    toast.error('Could not update. Please try again.');
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['posts'] }),
});
```

---

## Forms & Inputs

**Email not trimmed and lowercased on blur**
```tsx
<input
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  onBlur={() => setEmail((v) => v.trim().toLowerCase())}
  autoComplete="email"
/>
```

**Name fields reject apostrophes or Unicode**
```ts
// BAD
/^[A-Za-z]+$/

// GOOD: any letter in any language, combining marks, spaces, apostrophes, hyphens, periods
/^[\p{L}\p{M}][\p{L}\p{M}' .-]*$/u
```

**Missing `autocomplete` attributes**
| Field | Value |
|---|---|
| Full name | `name` |
| Email | `email` |
| Phone | `tel` |
| Login password | `current-password` |
| Signup or reset password | `new-password` |
| Street address | `address-line1` |
| Postal code | `postal-code` |

**Validation fires on every keystroke**
Validate on blur and on submit. With React Hook Form:
```ts
useForm({ mode: 'onBlur', reValidateMode: 'onChange' });
```

**Inline errors not accessible**
```tsx
<label htmlFor="email">Email</label>
<input
  id="email"
  aria-invalid={!!error}
  aria-describedby={error ? 'email-error' : undefined}
/>
{error && <p id="email-error" role="alert">{error}</p>}
```

**Submit disabled with no explanation**
Keep the button enabled (except while a request is in flight). On submit, show errors and focus the first invalid field:
```tsx
function onInvalid(errors: FieldErrors) {
  const first = Object.keys(errors)[0];
  document.getElementById(first)?.focus();
}
// React Hook Form: handleSubmit(onValid, onInvalid)
```

**Date inputs allow impossible dates**
```html
<input type="date" min="2026-01-01" max="2026-12-31" />
```
Compute `min` and `max` from the current date for fields such as date of birth or booking dates.

---

## Media & Uploads

**No client-side compression**
```ts
import imageCompression from 'browser-image-compression';

const compressed = await imageCompression(file, {
  maxSizeMB: 1,
  maxWidthOrHeight: 1920,
  useWebWorker: true,
});
```

**No MIME or size validation**
```ts
const ALLOWED = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_MB = 5;

if (!ALLOWED.includes(file.type)) return setError('Please choose a JPG, PNG, or WebP image.');
if (file.size > MAX_MB * 1024 * 1024) return setError(`File is too large. The limit is ${MAX_MB} MB.`);
```
Always validate type and size again on the server.

**No upload progress**
`fetch` does not report upload progress; use `XMLHttpRequest`:
```ts
const xhr = new XMLHttpRequest();
xhr.upload.onprogress = (e) => {
  if (e.lengthComputable) setProgress(Math.round((e.loaded / e.total) * 100));
};
xhr.open('POST', '/api/upload');
xhr.send(formData);
```

---

## State & Resilience

**Multi-step form state not persisted**
```ts
useEffect(() => {
  sessionStorage.setItem('signup-draft', JSON.stringify(values));
}, [values]);

const initial = JSON.parse(sessionStorage.getItem('signup-draft') ?? '{}');
```
Never persist passwords or card numbers. Clear the draft after a successful submit.

**Timestamps not UTC**
```ts
// Store
const createdAt = new Date().toISOString(); // UTC, ISO-8601

// Render in the user's timezone
new Intl.DateTimeFormat(undefined, { dateStyle: 'medium', timeStyle: 'short' }).format(new Date(createdAt));
```

**No offline banner or retry**
```tsx
function useOnline() {
  const [online, setOnline] = useState(true);
  useEffect(() => {
    const update = () => setOnline(navigator.onLine);
    update();
    window.addEventListener('online', update);
    window.addEventListener('offline', update);
    return () => {
      window.removeEventListener('online', update);
      window.removeEventListener('offline', update);
    };
  }, []);
  return online;
}

// In the layout
{!online && <div role="status">You are offline. Changes will be retried when you reconnect.</div>}
```
On reconnect, show a "Tap to Retry" button that re-runs the failed request.

---

## Onboarding, Help & Flows

**Onboarding missing Skip or step indicator**
```tsx
<p aria-live="polite">Step {step} of {total}</p>
<button type="button" onClick={finish}>Skip</button>
```

**FAQ accordion not accessible**
Native elements need no JavaScript and support the keyboard:
```html
<details>
  <summary>How do I reset my password?</summary>
  <p>Use the "Forgot password" link on the login page.</p>
</details>
```

**Password reset reveals whether an account exists**
```ts
// Always return the same message, whether or not the account exists
return Response.json({ message: 'If an account exists for this email, a reset link has been sent.' });
```

**Resend button without cooldown**
```tsx
const [cooldown, setCooldown] = useState(0);

useEffect(() => {
  if (cooldown <= 0) return;
  const id = setTimeout(() => setCooldown((c) => c - 1), 1000);
  return () => clearTimeout(id);
}, [cooldown]);

<button type="button" disabled={cooldown > 0} onClick={() => { resend(); setCooldown(60); }}>
  {cooldown > 0 ? `Resend in ${cooldown}s` : 'Resend email'}
</button>
```

**Console errors**
Add the lint rule that catches missing keys, then run the app and clear the browser console on each main page:
```bash
npm install --save-dev eslint-plugin-react
# .eslintrc: "react/jsx-key": "error"
```

---

## Accessibility & Visual

**Color contrast below WCAG AA**
Check pairs with https://webaim.org/resources/contrastchecker/ or the browser devtools contrast indicator. Targets: 4.5:1 for normal text, 3:1 for large text and UI components. Darken the text or lighten the background until the ratio passes, and update the design token so the fix applies everywhere.

**Missing focus styles**
```css
:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}
```
Never write `outline: none` without adding a replacement focus style.

**Inputs without labels**
```html
<label for="phone">Phone number</label>
<input id="phone" type="tel" autocomplete="tel" />
```
For an icon-only control, use `aria-label="Search"`. A placeholder is not a label.

**No custom 404 page**
Next.js App Router - create `app/not-found.tsx`:
```tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <main>
      <h1>Page not found</h1>
      <p>The page you are looking for does not exist or has moved.</p>
      <Link href="/">Back to the homepage</Link>
    </main>
  );
}
```
Laravel: create `resources/views/errors/404.blade.php` with the same content.