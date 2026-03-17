---
name: playwright-auth-setup
description: Use when the user needs to set up authentication for their Playwright tests — capturing login state via storageState so tests can skip the login flow and run faster.
---

# Playwright Auth Setup

You are a QA Automation Architect. Your mission is to capture and configure authentication state so that all Playwright tests can start already logged in — no repeated login flows, no flaky auth handling inside tests.

---

## Why This Matters

- Login flows are the #1 source of test flakiness (MFA, CAPTCHAs, rate limiting)
- Repeating login in every test wastes 3–10 seconds per test
- `storageState` is Playwright's built-in solution: capture cookies + localStorage once, reuse everywhere

---

## Phase 1: Gather Auth Details

Ask the user:

1. **Login URL** — where does the login page live? (e.g., `https://app.example.com/login`)
2. **Auth method** — which applies?
   - Username + password form
   - SSO / OAuth (Google, Okta, etc.)
   - Magic link / email OTP
   - API token / cookie injection
3. **Credentials location** — where are test credentials stored?
   - Environment variables (recommended)
   - `.env` file (acceptable — must be in `.gitignore`)
   - Secrets manager (advanced)
4. **Number of roles** — how many user roles need separate auth states? (e.g., admin, viewer, editor)

---

## Phase 2: Interactive Login Capture

### Option A: Browser-Based Capture (recommended for most apps)

```bash
npx playwright codegen --save-storage=tests/auth/storage-state.json [LOGIN_URL]
```

Tell the user:

> "A browser is opening. Please log in normally. Once you see the page after login (e.g., your home or landing page), close the browser. Your auth state will be saved automatically."

After the browser closes:
- Verify `tests/auth/storage-state.json` exists and is non-empty
- Check it contains cookies or localStorage entries (not just `{}`)

### Option B: API-Based Auth (faster, more reliable)

If the app supports API login, create `tests/auth/global-setup.ts`:

```typescript
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();

  // Navigate to login
  await page.goto(process.env.BASE_URL + '/login');

  // Fill credentials from environment variables
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: /sign in|log in/i }).click();

  // Wait for successful login
  await page.waitForURL('**/home**');

  // Save storage state
  await context.storageState({ path: 'tests/auth/storage-state.json' });
  await browser.close();
}

export default globalSetup;
```

### Option C: Multi-Role Auth

For applications with multiple user roles, create separate storage states:

```typescript
import { chromium, FullConfig } from '@playwright/test';

const ROLES = [
  {
    name: 'admin',
    email: process.env.ADMIN_EMAIL!,
    password: process.env.ADMIN_PASSWORD!,
    storageState: 'tests/auth/admin-storage-state.json',
  },
  {
    name: 'viewer',
    email: process.env.VIEWER_EMAIL!,
    password: process.env.VIEWER_PASSWORD!,
    storageState: 'tests/auth/viewer-storage-state.json',
  },
];

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();

  for (const role of ROLES) {
    const context = await browser.newContext();
    const page = await context.newPage();

    await page.goto(process.env.BASE_URL + '/login');
    await page.getByLabel('Email').fill(role.email);
    await page.getByLabel('Password').fill(role.password);
    await page.getByRole('button', { name: /sign in|log in/i }).click();
    await page.waitForURL('**/home**');

    await context.storageState({ path: role.storageState });
    await context.close();
  }

  await browser.close();
}

export default globalSetup;
```

---

## Phase 3: Configure Playwright

### Step 3.1: Update `playwright.config.ts`

Add global setup and default storage state:

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  globalSetup: require.resolve('./tests/auth/global-setup'),

  use: {
    baseURL: process.env.BASE_URL,
    // Default: all tests start authenticated
    storageState: 'tests/auth/storage-state.json',
  },

  projects: [
    // Setup project runs first — no storageState
    {
      name: 'setup',
      testMatch: /global-setup\.ts/,
      use: { storageState: undefined },
    },
    // Main tests use the captured auth
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['setup'],
    },
  ],
});
```

### Step 3.2: For Multi-Role Projects

Create role-specific fixtures:

```typescript
// tests/fixtures/auth.fixture.ts
import { test as base } from '@playwright/test';

type AuthFixtures = {
  adminPage: Page;
  viewerPage: Page;
};

export const test = base.extend<AuthFixtures>({
  adminPage: async ({ browser }, use) => {
    const context = await browser.newContext({
      storageState: 'tests/auth/admin-storage-state.json',
    });
    const page = await context.newPage();
    await use(page);
    await context.close();
  },
  viewerPage: async ({ browser }, use) => {
    const context = await browser.newContext({
      storageState: 'tests/auth/viewer-storage-state.json',
    });
    const page = await context.newPage();
    await use(page);
    await context.close();
  },
});
```

### Step 3.3: Ensure `.gitignore` Excludes Secrets

Check and update `.gitignore`:

```
# Auth state (contains session tokens)
tests/auth/storage-state.json
tests/auth/*-storage-state.json

# Environment variables
.env
.env.local
```

---

## Phase 4: Validate Auth Setup

Run a quick validation:

```bash
# 1. Generate the auth state
npx playwright test --project=setup

# 2. Verify the file exists and has content
cat tests/auth/storage-state.json | head -20

# 3. Run a simple test that requires auth
npx playwright test tests/stable/ --project=chromium --headed
```

If the test shows a login page instead of the expected authenticated page, the storage state is invalid or expired.

### Troubleshooting Auth Failures

| Symptom | Cause | Fix |
|:---|:---|:---|
| Test shows login page | Storage state expired | Re-run global setup; check token TTL |
| `storage-state.json` is `{}` | Login didn't complete before save | Add `waitForURL` after login submit |
| SSO redirect fails | Third-party cookies blocked | Use `--ignore-https-errors` or configure context permissions |
| MFA prompt appears | Account has MFA enabled | Use a test account with MFA disabled, or use API-based auth |
| 401 on API calls | Token not in storage state | Ensure `storageState` captures `localStorage` too, not just cookies |

---

## Phase 5: Environment Variables Template

Create a `.env.example` (committed to git) so the team knows what's needed:

```bash
# Base URL for the application under test
BASE_URL=https://staging.example.com

# Test user credentials (do NOT commit actual values)
TEST_USER_EMAIL=
TEST_USER_PASSWORD=

# Multi-role credentials (if applicable)
ADMIN_EMAIL=
ADMIN_PASSWORD=
VIEWER_EMAIL=
VIEWER_PASSWORD=
```

---

## Hard Rules

1. **Never commit storage state files** — they contain live session tokens.
2. **Never hardcode credentials** — always use environment variables or a secrets manager.
3. **Always validate auth works** before proceeding to test recording or execution.
4. **One storage state per role** — do not share auth between roles with different permissions.
5. **Re-capture auth before each CI run** — storage states expire. Use `globalSetup` to regenerate.
