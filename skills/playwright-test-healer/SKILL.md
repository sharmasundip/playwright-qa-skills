---
name: playwright-test-healer
description: Use when a Playwright test is failing or flaky and needs to be diagnosed, repaired, and verified as stable before it can be promoted to the validated test suite.
---

# Playwright Test Healer

You are a Principal SDET. Your mission is to transform failing or flaky Playwright tests into stable, production-grade assets. You do not patch symptoms — you find root causes and fix them durably.

---

## Prerequisites

This skill uses the following tools during diagnosis:

- **`npx playwright test`** — to run tests and capture error output
- **Playwright trace viewer** — traces are captured on failure when `trace: 'on-first-retry'` is configured
- **Test HTML report** — `npx playwright show-report` for visual failure analysis
- **Terminal output** — error messages, stack traces, and assertion diffs from test runs

If `playwright.config.ts` has `trace: 'on-first-retry'`, the trace file provides DOM snapshots, network logs, and action timeline — use it as the primary diagnostic source.

---

## The Healing Diagnostic Matrix

Before any code change, categorize the failure against these six vectors:

| Vector | Root Cause | Healing Strategy | Frequency |
|:---|:---|:---|:---:|
| **Timing & Sync** | Hydration race, async lag | Replace `waitForTimeout` with `expect().toPass()` or `waitForResponse` | ~30% |
| **Selector Decay** | DOM or attribute changes | Use trace viewer for semantic re-mapping (aria-roles, testIds) | ~28% |
| **Test Data** | Expired state, missing fixtures | Detect 4xx/5xx; refresh auth state or re-seed data | ~14% |
| **Interaction** | Hidden or obscured elements | Detect overlays; use `force: true` or dismiss blockers first | ~10% |
| **Visuals** | Dynamic content in screenshots | Mask timestamps, IDs, and animated regions in visual comparisons | ~10% |
| **Runtime/Env** | App crashes, config mismatch | Wrap flaky blocks in `test.step` retries; check env config | ~8% |

---

## Standard Operating Procedure

### Step 1: Initial Run and Context Capture

```bash
npx playwright test [file-path] --project=chromium --reporter=list
```

If the test fails:
- Read the error message and stack trace carefully
- Check for a trace file in `test-results/` — if present, examine it:
  ```bash
  npx playwright show-trace test-results/[test-name]-chromium/trace.zip
  ```
- Look for failed API calls in the trace (4xx/5xx responses)
- Read the error message carefully before touching any code

### Step 2: Root Cause Analysis

Map the failure to the diagnostic matrix:

| Error Pattern | Vector | First Action |
|:---|:---|:---|
| "element not attached" | Timing | Use polling assertion (`expect().toPass()`) |
| "locator resolved to hidden element" | Interaction | Check for overlay/modal in trace |
| "strict mode violation" | Selector Decay | Locator is too broad — add `{ exact: true }` or narrow with filters |
| "expect received 404/500" | Test Data | Check auth state validity, re-run `global-setup` |
| "timeout exceeded" | Timing or Selector | Check trace — is the element missing from DOM or just slow? |
| "Target closed" / "browser closed" | Runtime/Env | App crashed — check server logs if accessible |
| "net::ERR_" | Runtime/Env | Network issue — check `BASE_URL` and app availability |

**Network-aware RCA**: If an API returned 4xx or 5xx, classify as Test Data or Runtime — not a selector issue.

**Auth-aware RCA**: If you see a redirect to `/login` or a 401 response, the storage state is likely expired. Re-run the `playwright-auth-setup` skill before continuing.

### Step 3: Apply the Fix

Use this locator priority when rewriting selectors:

```
getByTestId > getByRole > getByLabel > getByPlaceholder > getByText
```

**Timing fixes:**
```typescript
// Bad
await page.waitForTimeout(2000);

// Good — poll until condition is true
await expect(async () => {
  await expect(page.getByTestId('status-badge')).toHaveText('Active');
}).toPass({ timeout: 10_000 });

// Good — wait for specific network response
const [response] = await Promise.all([
  page.waitForResponse(res => res.url().includes('/api/records') && res.status() === 200),
  page.getByRole('button', { name: 'Submit' }).click(),
]);

// Good — wait for navigation
await Promise.all([
  page.waitForURL('**/records/*'),
  page.getByRole('link', { name: 'View Details' }).click(),
]);
```

**Selector fixes:**
```typescript
// Bad
page.locator('.submit-button')
page.locator('xpath=//button[3]')

// Good
page.getByRole('button', { name: 'Submit' })
page.getByTestId('submit-btn')

// Strict mode fix — too many matches
// Bad: page.getByRole('button', { name: 'Save' }) — matches 3 buttons
// Good: narrow the scope
page.getByTestId('edit-form').getByRole('button', { name: 'Save' })
```

**Overlay/obstruction fix:**
```typescript
// Dismiss a cookie banner or modal before interacting
const overlay = page.getByRole('dialog');
if (await overlay.isVisible()) {
  await overlay.getByRole('button', { name: /close|dismiss|accept/i }).click();
}
```

**Dynamic string matching:**
```typescript
// Use regex for IDs or auto-generated values
page.getByRole('heading', { name: /REC-\d+/ })
```

**Auth state refresh:**
```bash
# If 401s or login redirects appear, re-capture auth
npx playwright test --project=setup
# Then re-run the failing test
npx playwright test [file-path] --project=chromium
```

### Step 4: Verify — The Stability Gate

<IMPORTANT>
After each fix, run the test. Do NOT declare it stable after one pass.

Apply the 3/3 Pass Rule: the test must pass 3 consecutive times before it is considered stable.
</IMPORTANT>

```bash
# Run 3 times consecutively — all must pass
npx playwright test [file] --project=chromium --repeat-each=3
```

If `--repeat-each` is not available, run manually 3 times:

```bash
npx playwright test [file] --project=chromium && \
npx playwright test [file] --project=chromium && \
npx playwright test [file] --project=chromium
```

**Progress reporting** — after each iteration, tell the user:

> "Iteration [N]/5: [what was found and fixed]. Running stability check..."

Repeat the Investigate → RCA → Remediate → Verify loop up to **5 iterations**.

After 5 iterations with no stable result, invoke the circuit breaker (see Step 5).

---

## Step 5: Atomic Migration

**If stable (3/3 passes):**
```bash
mkdir -p tests/stable
mv tests/generated/[module]/specs/[test-name].spec.ts tests/stable/[test-name].spec.ts
```

**If unresolved after 5 iterations:**
1. Add `test.fixme()` at the top of the failing test
2. Add an RCA comment block:
   ```typescript
   // HEALER RCA: Attempted 5 iterations. Root cause: [vector].
   // Tried: [list of strategies attempted].
   // Unresolved: [what remains broken and why].
   // Suggested next steps: [what a human should try].
   ```
3. Move to: `tests/quarantine/[test-name].spec.ts`
   ```bash
   mkdir -p tests/quarantine
   mv tests/generated/[module]/specs/[test-name].spec.ts tests/quarantine/[test-name].spec.ts
   ```

---

## Cross-Browser Healing (Optional)

If the user's config includes multiple browser projects, after achieving stability on Chromium, optionally verify on other browsers:

```bash
npx playwright test [file] --project=firefox
npx playwright test [file] --project=webkit
```

Common cross-browser issues:
- WebKit handles `click()` on overlapping elements differently — may need `force: true`
- Firefox has stricter CORS handling — API interception patterns may differ
- Safari/WebKit doesn't support certain CSS selectors in `locator()`

Only flag cross-browser issues if they appear — do not preemptively add workarounds.

---

## Hard Rules

1. **Never change test intent** — do not remove assertions or steps to get a green pass.
2. **Trace before locator fixes** — never guess at a new selector without examining the failure trace or DOM state.
3. **No brittle fixes** — avoid `nth(x)`, deep CSS paths, or `force: true` without understanding why the element is obscured.
4. **3/3 pass rule is mandatory** — one passing run is not stability. Use `--repeat-each=3` when possible.
5. **5 iterations max** — if not fixed by then, escalate to `needs-review` with full RCA notes.
6. **Atomic moves** — always use `mv` (not copy), so the source location is cleaned up.
7. **Check auth first** — if you see 401s or login redirects, fix auth before chasing selector issues.
8. **Report progress** — tell the user what you're doing at each iteration. Silence is confusing.
