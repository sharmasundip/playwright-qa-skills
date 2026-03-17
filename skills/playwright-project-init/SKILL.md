---
name: playwright-project-init
description: Use when a QA team needs to set up a new Playwright test project from scratch — scaffolding the directory structure, config, dependencies, and CI pipeline in one command.
---

# Playwright Project Init

You are a QA Infrastructure Architect. Your mission is to scaffold a complete, production-ready Playwright test project so QA teams can start recording and running tests immediately — zero manual configuration required.

---

## Phase 1: Pre-Flight Check

Before scaffolding, determine the current state:

### Step 1.1: Check for Existing Project

```bash
ls package.json playwright.config.ts 2>/dev/null
```

| State | Action |
|:---|:---|
| No `package.json` | Full scaffolding — create everything |
| `package.json` exists, no Playwright | Add Playwright to existing project |
| `playwright.config.ts` exists | Enhancement mode — add only missing pieces |

### Step 1.2: Gather Inputs

Ask the user:

1. **Application URL** — the base URL of the app being tested (e.g., `https://staging.example.com`)
2. **Does the app require login?** (yes/no) — determines whether auth setup is included
3. **Which browsers?** — default: Chromium only for speed. Options: Chromium, Firefox, WebKit, or all three
4. **CI provider** (optional) — GitHub Actions, GitLab CI, Jenkins, or none for now

---

## Phase 2: Scaffold Directory Structure

Create the full test project tree:

```bash
mkdir -p tests/{recordings,recordings/reports,generated,stable,quarantine,auth,fixtures,api-blueprints}
```

Final structure:

```
tests/
  recordings/                   ← recorded tests (source of truth, read-only)
    reports/                    ← review scores and feedback
  generated/                    ← refactored POM tests (organized by module)
  stable/                       ← tests that passed 3/3 stability gate
  quarantine/                   ← tests that couldn't be healed (with RCA notes)
  auth/                         ← storage state files (.gitignored)
  fixtures/                     ← shared fixtures (auth, HAR, etc.)
  api-blueprints/               ← HAR captures and API blueprints
```

---

## Phase 3: Install Dependencies

```bash
# Initialize package.json if missing
npm init -y 2>/dev/null

# Install Playwright with browsers
npm install -D @playwright/test

# Install browsers (Chromium by default, or all if user requested)
npx playwright install chromium
# or: npx playwright install
```

### Optional Dependencies

Ask the user if they need any of these:

| Package | Purpose | Install When |
|:---|:---|:---|
| `dotenv` | Load `.env` files for credentials | App requires auth |
| `@playwright/test` (already installed) | Core framework | Always |
| `allure-playwright` | Rich HTML test reports | User wants detailed reports |

---

## Phase 4: Generate Configuration Files

### Step 4.1: `playwright.config.ts`

```typescript
import { defineConfig, devices } from '@playwright/test';
import dotenv from 'dotenv';
import path from 'path';

// Load environment variables from .env file
dotenv.config({ path: path.resolve(__dirname, '.env') });

export default defineConfig({
  // Where to find test files
  testDir: './tests',
  testMatch: ['stable/**/*.spec.ts', 'generated/**/specs/**/*.spec.ts'],

  // Run tests in parallel
  fullyParallel: true,

  // Fail CI if test.only is left in source code
  forbidOnly: !!process.env.CI,

  // Retry failed tests (2 retries in CI, 0 locally)
  retries: process.env.CI ? 2 : 0,

  // Limit parallel workers in CI to avoid resource issues
  workers: process.env.CI ? 2 : undefined,

  // Reporter configuration
  reporter: process.env.CI
    ? [['html', { open: 'never' }], ['json', { outputFile: 'test-results/results.json' }]]
    : [['html', { open: 'on-failure' }]],

  // Shared settings for all projects
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',

    // Capture trace on first retry for debugging
    trace: 'on-first-retry',

    // Capture screenshot on failure
    screenshot: 'only-on-failure',

    // Capture video on first retry
    video: 'on-first-retry',

    // Default timeout per action (click, fill, etc.)
    actionTimeout: 10_000,

    // Default navigation timeout
    navigationTimeout: 30_000,
  },

  // Browser projects
  projects: [
    // Auth setup — runs before all test projects
    {
      name: 'setup',
      testMatch: /global-setup\.ts/,
      use: { storageState: undefined },
    },

    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'tests/auth/storage-state.json',
      },
      dependencies: ['setup'],
    },

    // Uncomment to enable additional browsers:
    // {
    //   name: 'firefox',
    //   use: {
    //     ...devices['Desktop Firefox'],
    //     storageState: 'tests/auth/storage-state.json',
    //   },
    //   dependencies: ['setup'],
    // },
    // {
    //   name: 'webkit',
    //   use: {
    //     ...devices['Desktop Safari'],
    //     storageState: 'tests/auth/storage-state.json',
    //   },
    //   dependencies: ['setup'],
    // },
  ],

  // Global test timeout
  timeout: 60_000,

  // Expect timeout
  expect: {
    timeout: 10_000,
  },
});
```

**If the user said no login required**, remove the `setup` project and `storageState` lines.

### Step 4.2: `.env.example`

```bash
# Application under test
BASE_URL=https://staging.example.com

# Test credentials (copy to .env and fill in values)
TEST_USER_EMAIL=
TEST_USER_PASSWORD=

# CI flag (set automatically in CI pipelines)
# CI=true
```

### Step 4.3: `.gitignore` additions

Append to existing `.gitignore` or create one:

```
# Playwright
tests/auth/storage-state.json
tests/auth/*-storage-state.json
test-results/
playwright-report/
blob-report/

# Environment
.env
.env.local

# OS
.DS_Store
```

### Step 4.4: `tsconfig.json` (if missing)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "outDir": "./dist",
    "rootDir": "."
  },
  "include": ["tests/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## Phase 5: Generate Starter Files

### Step 5.1: Global Auth Setup (if login required)

Create `tests/auth/global-setup.ts`:

```typescript
import { chromium, FullConfig } from '@playwright/test';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();

  await page.goto(process.env.BASE_URL + '/login');
  await page.getByLabel('Email').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: /sign in|log in/i }).click();
  await page.waitForURL('**/home**');

  await context.storageState({ path: 'tests/auth/storage-state.json' });
  await browser.close();
}

export default globalSetup;
```

Tell the user:

> "I've created a starter login script. You will need to update the selectors (`getByLabel('Email')`, etc.) to match your app's actual login form. Run the auth setup skill to capture this interactively."

### Step 5.2: Base Fixture

Create `tests/fixtures/base.fixture.ts`:

```typescript
import { test as base } from '@playwright/test';

// Extend this fixture to add page objects for each module
export const test = base.extend({
  // Example: uncomment and add your page objects here
  // recordPage: async ({ page }, use) => {
  //   await use(new RecordPage(page));
  // },
});

export { expect } from '@playwright/test';
```

### Step 5.3: Smoke Test

Create `tests/stable/smoke.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';

test.describe('Smoke Test', () => {
  test('application loads successfully', async ({ page }) => {
    await page.goto('/');
    await expect(page).not.toHaveTitle(/error|404|500/i);
    await expect(page.locator('body')).toBeVisible();
  });
});
```

---

## Phase 6: CI/CD Setup (If Requested)

### GitHub Actions

Create `.github/workflows/playwright.yml`:

```yaml
name: Playwright Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run Playwright tests
        run: npx playwright test
        env:
          BASE_URL: ${{ secrets.BASE_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
          CI: true

      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 14

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: test-results
          path: test-results/
          retention-days: 7
```

Tell the user:

> "Add your `BASE_URL`, `TEST_USER_EMAIL`, and `TEST_USER_PASSWORD` as GitHub repository secrets under Settings → Secrets and variables → Actions."

### GitLab CI

Create `.gitlab-ci.yml`:

```yaml
stages:
  - test

playwright:
  stage: test
  image: mcr.microsoft.com/playwright:v1.50.0-noble
  script:
    - npm ci
    - npx playwright test
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    expire_in: 7 days
  variables:
    BASE_URL: $BASE_URL
    TEST_USER_EMAIL: $TEST_USER_EMAIL
    TEST_USER_PASSWORD: $TEST_USER_PASSWORD
    CI: "true"
```

---

## Phase 7: Validate Setup

Run these checks to confirm everything works:

```bash
# 1. Verify Playwright is installed
npx playwright --version

# 2. Run the smoke test (skip auth if not configured yet)
npx playwright test tests/stable/smoke.spec.ts --project=chromium

# 3. Open the HTML report
npx playwright show-report
```

### Deliver the Summary

```
Project Setup Complete
─────────────────────────────────────────
Directory structure:  ✓ created
Dependencies:         ✓ installed
Config:               ✓ playwright.config.ts
Auth setup:           [✓ created / ⊘ not required]
CI pipeline:          [✓ GitHub Actions / ✓ GitLab CI / ⊘ skipped]
Smoke test:           [✓ passing / ✗ needs BASE_URL]
─────────────────────────────────────────
Next step: Run the auth setup skill or start recording your first test!
```

---

## Hard Rules

1. **Never overwrite existing files** — always check first and merge or extend.
2. **Never commit credentials** — `.env` and storage state files must be in `.gitignore`.
3. **Always validate the setup** — run the smoke test before declaring success.
4. **Keep config minimal** — only enable what the user asked for. Commented-out options are fine for discoverability.
5. **Use `process.env`** for all environment-specific values — never hardcode URLs or credentials.
