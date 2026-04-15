# Domain event scan: grep patterns for Step 6

Step 6 of the installer scans the target SPA for likely event hook points and proposes tracking calls. This file is the pattern catalog.

## How the scan runs

1. Scope: grep **only inside the currently-targeted SPA's `src/` directory**. Do not scan other SPAs, node_modules, or tests.
2. For each pattern below, run the grep and collect `file:line` matches.
3. Deduplicate: one suggestion per unique `file:line`. If multiple patterns match the same location, pick the highest-priority pattern.
4. Rank by confidence (P0 patterns first, then P1, then P2).
5. Present interactively, one at a time, with y/n/rename.

## Pattern catalog

### P0 — Form submissions (highest confidence)

| Grep | Signal | Suggested event |
|---|---|---|
| `onSubmit=\{` on JSX `<form>` tags | User-initiated form submission | `trackLead()` + `trackEvent('form_submitted')` |
| `handleSubmit\s*=` or `const handleSubmit` near `await fetch\|await supabase` | Form handler with network call | `trackEvent('<form_name>_submitted')` |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "onSubmit=\{" src/
```

**When to use `trackLead()` vs `trackEvent('form_submitted')`**: `Lead` is a Meta standard event reserved for contact/signup forms where the user intent is "I'm interested." For every form, propose both if the form is in a top-of-funnel path (welcome screen, contact page); propose only `trackEvent` for utility forms (profile edit, settings).

**Auto-skip**: forms inside `/admin/` or `/settings/` paths — these are administration, not conversion.

### P0 — Checkout-related calls

| Grep | Signal | Action |
|---|---|---|
| `supabase.functions.invoke\(['"]create-checkout` | Stripe checkout initiation | **Already handled by Step 4 Stripe patches** — do not propose |
| `window.location.href.*stripe.*checkout.sessions` | Direct Stripe redirect | Propose `trackInitiateCheckout(eventId)` before the redirect |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "create-checkout|stripe.*checkout.sessions" src/
```

Skip matches inside `src/lib/` (library code, not user interactions).

### P1 — CTA button clicks

| Grep | Signal | Suggested event |
|---|---|---|
| `onClick=\{[^}]*\}` on `<button>` with text matching `/sign ?up\|get started\|start\|join\|subscribe\|continue/i` | CTA click | `trackEvent('cta_click', { cta: '<button_text>' })` |

**Ripgrep commands** (two-step: find buttons first, then filter by surrounding text):
```bash
# Step 1: find all button onClick handlers
rg -n --type=tsx "onClick=\{[^}]*\}" src/ | rg "button"

# Step 2: for each match, read 3 lines around it and check for CTA text
# (the installer does this in Claude rather than shell)
```

**Renaming**: ask the user for a more descriptive name than `cta_click` if the button's role is specific ("book_dinner", "upgrade_plan"). Default to `cta_click` if unclear.

### P1 — Authentication events

| Grep | Signal | Suggested event |
|---|---|---|
| `supabase.auth.signInWithOtp` | Magic-link login started | `trackEvent('login_started')` before, `trackEvent('login_completed')` after |
| `supabase.auth.signUp\(` | Password-based signup | `trackLead()` + `trackEvent('signup_completed')` |
| `supabase.auth.signInAnonymously` | Anonymous session creation | Usually too-early-in-funnel — skip unless user asks |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "supabase\.auth\.(signIn|signUp|signOut)" src/
```

### P1 — Funnel step navigation

| Grep | Signal | Suggested event |
|---|---|---|
| `navigate\(['"]/step/` or `useNavigate` call inside an onClick | Multi-step funnel navigation | `trackEvent('step_completed', { step: '<step_id>' })` |
| `setStep\s*\(` or `onNext\s*\(` | Local step state advance | Same as above |

**Ripgrep command:**
```bash
rg -n --type=tsx "navigate\(['\"]/step/|setStep\(|onNext\(" src/
```

### P2 — Engagement events (lower confidence, offer cautiously)

| Grep | Signal | Suggested event |
|---|---|---|
| `setXEnabled\s*\(\s*true` | Toggle flipped on | `trackEvent('feature_enabled', { feature: '<name>' })` |
| `\.update\(\s*\{[^}]*opted_in:\s*true` | Database opt-in write | `trackEvent('opted_in')` |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "opted_in|opt_in" src/
```

These P2 patterns often produce false positives — skip by default unless the user confirms.

### Negative patterns (auto-skip, never propose)

- `trackEvent\|trackLead\|trackPixelEvent` — the wrapper is already present at this site, don't double-track
- Paths containing `/test/`, `/tests/`, `__tests__`, `.test.`, `.spec.` — test files, not production
- Paths containing `/admin/` — administration, not conversion
- Files starting with `use` and returning hooks (`useFoo.ts`) — utility hooks, tracking belongs at the call site

## Suggestion presentation format

After running all patterns, present results one at a time:

```
Found <N> candidate event hook points across <M> files.
For each, I'll show the code and propose an event name.
You say y/n/rename. Nothing is edited until you confirm.

[1/<N>] <relative/path/to/file.tsx>:<line>
        <code context, 2-3 lines>
        Suggested: <tracking code to insert>
        y / n / rename: _
```

### On `y` — wire the event

Anchored insertion rules:

- **Inside an `onSubmit` handler**: insert the track call as the **first statement** of the handler, before any validation or async work. If the handler is an inline arrow function (`onSubmit={(e) => { ... }}`), extract it to a named handler first. If the handler is already named, add the call at the top.
- **Inside an `onClick` handler**: same — first statement, even before `e.preventDefault()`.
- **Inside an auth call chain**: `trackEvent('login_started')` before the `await signInWithOtp(...)`, `trackEvent('login_completed')` after it resolves successfully.
- **Inside a navigation call**: `trackEvent('step_completed', { step })` on the line above the `navigate(...)` call.

**Always add the import** at the top of the file if not already present:

```typescript
import { trackEvent } from './lib/analytics'
// and/or:
import { trackLead, trackInitiateCheckout, trackPixelEvent } from './lib/meta-pixel'
```

Path depth depends on the file location — use `./`, `../`, or `../../` as needed.

### On `rename` — customise the event name

Prompt: `"Event name (default: <suggested>):"`. User types a new name (or hits enter to accept). Then wire it with the new name. Use GA4 conventions: `snake_case`, verb-noun, no spaces.

### On `n` — skip

No edit. Move to the next suggestion. The `n` option is the safe default for unsure matches.

### Final summary

At the end:

```
Wired <K> events across <M> files. Skipped <L>. Renamed <R>.
```

## Event naming conventions

| Standard Meta events (use capitalised) | Custom events (use snake_case) |
|---|---|
| `Lead` | `form_submitted` |
| `InitiateCheckout` | `cta_click` |
| `Purchase` | `step_completed` |
| `CompleteRegistration` | `login_started` |
| `ViewContent` | `login_completed` |
| `AddToCart` | `feature_enabled` |

Standard Meta events go through **both** `trackPixelEvent` (with capitalised name) and `trackEvent` (with snake_case equivalent) so both backends see the conversion. Custom events go through `trackEvent` only unless the user explicitly asks for pixel coverage.
