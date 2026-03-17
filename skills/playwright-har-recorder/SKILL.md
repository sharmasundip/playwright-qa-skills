---
name: playwright-har-recorder
description: Use when you need to capture, clean, and parameterize the network traffic from a Playwright test into reusable API blueprints that can be replayed across environments.
---

# Playwright HAR Recorder

You are a Principal API Architect. Your mission is to transform Playwright network captures into environment-agnostic API blueprints — clean, parameterized, replayable API sequences.

---

## Phase 0: Gather Inputs

Before doing anything, ask the user:

1. **Which test to record?** — path to the validated spec file (e.g., `tests/stable/create-record.spec.ts`), or a module name to list options from `tests/stable/`
2. **What API domain or URL pattern should be captured?**
   > Example: `api.example.com`, `/api/v1/`, `example.com/api`

   This is required — do not proceed without it. This filter determines which network calls are captured.

3. **What domains should be excluded?** (optional)
   > Common defaults to exclude: analytics, monitoring, CDN, error tracking (e.g., `sentry.io`, `amplitude.com`, `pendo.io`, `cdn.example.com`)
   > The user can confirm the defaults or add their own.

4. **Will this blueprint be used for mock testing?** (yes / no)
   > If yes, a route-handler replay file will be generated in Phase 6.

Save: `API_DOMAIN=[user-provided pattern]`, `MODULE=[derived from spec path]`, `SPEC_NAME=[spec filename without extension]`.

---

## Phase 1: Capture

HAR recording requires a fixture — Playwright CLI does not support `--save-har` directly.

### Step 1.1: Ensure HAR Fixture Exists

Check for `tests/fixtures/har.fixture.ts`. If missing, create it:

```typescript
import { test as base } from '@playwright/test';
import path from 'path';

export const test = base.extend({
  context: async ({ browser }, use, testInfo) => {
    const moduleName = path.basename(path.dirname(testInfo.file));
    const specName = path.basename(testInfo.file, '.spec.ts');
    const context = await browser.newContext({
      recordHar: {
        path: `tests/api-blueprints/raw/${moduleName}/${specName}.har`,
        urlFilter: new RegExp(process.env.HAR_URL_FILTER || '.*'),
      },
    });
    await use(context);
    await context.close(); // HAR saved on close
  },
});

export { expect } from '@playwright/test';
```

Set `HAR_URL_FILTER` to the user's domain pattern before running.

### Step 1.2: Run the Capture

```bash
HAR_URL_FILTER="[user-provided pattern]" npx playwright test [spec-path] --project=chromium
```

Use `--headed` if the test requires user interaction or MFA during the first capture.

### Step 1.3: Failure Handling

If the test fails during capture:
1. Report the error to the user
2. Suggest running the `playwright-test-healer` skill first to stabilize the test
3. Do NOT proceed with a partial or incomplete HAR

### Step 1.4: Verify Capture

After successful capture, verify the HAR file:
- File exists and is non-empty
- Contains entries matching the API domain filter
- Is valid JSON

```bash
# Quick validation
node -e "const h = require('./tests/api-blueprints/raw/[module]/[spec-name].har'); console.log('Entries:', h.log.entries.length)"
```

---

## Phase 2: Scrub & Redact

### Step 2.1: Domain Filtering

Keep only requests matching the user-provided domain pattern. Remove:
- Requests to excluded analytics/monitoring domains
- `OPTIONS` preflight requests
- Static assets: `.js`, `.css`, `.svg`, `.png`, `.jpg`, `.woff2`, `.ico`, `.map`
- Requests with non-JSON content types that aren't relevant to the flow

### Step 2.2: Header Cleanup

Strip non-functional headers. Keep only:
- `Authorization`
- `Content-Type`
- `Accept`
- Any custom API headers the user's app requires (ask if unsure)

Remove: browser fingerprinting headers, CDN headers, tracing IDs, user-agent strings.

### Step 2.3: Security Redaction

Replace sensitive values with named placeholders:

| Pattern | Replacement |
|:---|:---|
| `Authorization: Bearer <token>` | `{{AUTH_TOKEN}}` |
| `X-API-Key: <value>` | `{{API_KEY}}` |
| User IDs / account IDs | `{{USER_ID}}` |
| Tenant or org IDs | `{{TENANT_ID}}` |
| Entity IDs from POST responses | `{{ENTITY_ID}}` |
| Date strings | `{{TODAY}}` or `{{DATE}}` |
| Phone numbers, SSNs, card numbers | `{{REDACTED}}` |
| Email addresses | `{{USER_EMAIL}}` |

---

## Phase 3: Trace-Chain (Dependency Graphing)

Build a dependency graph across all captured requests. For each request in sequence:

1. **Scan response bodies** for IDs or values created by `POST`/`PATCH` (UUIDs, entity IDs, reference numbers, etc.)
2. **Search forward** through subsequent requests — headers, URL paths, query params, and bodies — for that exact value
3. **Replace** every hardcoded occurrence with a named variable: `{{recordId}}`, `{{itemId}}`, etc.
4. **Define a capture rule** with JSONPath on the originating response

### Dependency Map Format

```json
{
  "variables": [
    {
      "name": "recordId",
      "source": {
        "step": 1,
        "method": "POST",
        "url": "/api/records",
        "jsonPath": "$.data.id"
      },
      "used_in": [
        { "step": 3, "location": "url", "pattern": "/api/records/{{recordId}}" },
        { "step": 5, "location": "body", "pattern": "$.recordId" }
      ]
    }
  ]
}
```

Common patterns to check for in any application:
- Entity IDs from creation responses (UUIDs, numeric IDs)
- Session tokens or CSRF tokens
- Pagination cursors
- Date values from server responses reused in subsequent requests

---

## Phase 4: Parameterize

Generate a companion `variables.json`:

```json
{
  "source_spec": "tests/stable/[spec-name].spec.ts",
  "api_domain": "[user-provided domain pattern]",
  "static": {
    "BASE_URL": "",
    "TENANT_ID": ""
  },
  "dynamic": {
    "recordId": { "capture": "$.data.id", "from_step": 1 },
    "itemId": { "capture": "$.data.itemId", "from_step": 2 }
  },
  "temporal": {
    "TODAY": "{{TODAY}}",
    "TODAY_PLUS_1": "{{TODAY_PLUS_1}}"
  }
}
```

Leave `static` values empty — they are filled per environment at runtime.

---

## Phase 5: Validate

Run three gates before saving. Fail = do not proceed, fix the issue first.

| Gate | Check | Fail Action |
|:---|:---|:---|
| **Idempotency** | Can this sequence run twice without manual data cleanup? | Flag non-idempotent calls, suggest cleanup/teardown hooks |
| **Header minimalism** | Does each request work with only the kept headers? | Strip surplus headers and verify the test still passes |
| **Variable completeness** | Does every `{{variable}}` in a request have a capture rule in a prior response? | List orphaned variables and trace their origin |

---

## Phase 6: Save & Catalog

1. **Blueprint** → `tests/api-blueprints/[module]/[spec-name].har.json`
2. **Variables** → `tests/api-blueprints/[module]/[spec-name].variables.json`
3. **Dependency map** → `tests/api-blueprints/[module]/[spec-name].deps.json`
4. **Raw HAR archive** → `tests/api-blueprints/raw/[module]/[spec-name].har` (keep for debugging)

### Large HAR Optimization

If the captured HAR exceeds 5MB:
- Identify the first and last business-relevant API call
- Discard all entries outside the transaction window
- Log the trimmed entry count in the blueprint metadata

### Mock Replay File (If Requested)

If the user said yes to mock testing in Phase 0, generate a route handler:

File: `tests/api-blueprints/[module]/[spec-name].mock.ts`

```typescript
import { Page } from '@playwright/test';
import blueprint from './[spec-name].har.json';

/**
 * Replays the captured API responses as mocks.
 * Use this to run tests without a live backend.
 *
 * Usage in test:
 *   await setupMocks(page);
 *   // ... test proceeds using mocked responses
 */
export async function setupMocks(page: Page): Promise<void> {
  for (const entry of blueprint.log.entries) {
    const url = entry.request.url;
    const method = entry.request.method;

    await page.route(
      (route) => route.request().url().includes(url) && route.request().method() === method,
      (route) => {
        route.fulfill({
          status: entry.response.status,
          contentType: entry.response.content.mimeType,
          body: entry.response.content.text,
        });
      }
    );
  }
}
```

Tell the user:

> "I've generated a mock replay file. You can import `setupMocks` in any test to run against recorded API responses instead of a live server. This is useful for offline testing, CI pipelines, or testing edge cases."

---

## Phase 7: Environment Switching Guide

After saving, provide the user with environment switching instructions:

```
API Blueprint Saved
─────────────────────────────────────────
Blueprint:      tests/api-blueprints/[module]/[spec-name].har.json
Variables:      tests/api-blueprints/[module]/[spec-name].variables.json
Dependencies:   tests/api-blueprints/[module]/[spec-name].deps.json
Mock replay:    tests/api-blueprints/[module]/[spec-name].mock.ts  ← if requested
─────────────────────────────────────────

To use in a different environment:
1. Update the static variables in [spec-name].variables.json:
   - BASE_URL → your target environment URL
   - TENANT_ID → your target tenant
2. Dynamic variables (recordId, etc.) are captured at runtime — no changes needed
3. Temporal variables (TODAY, etc.) are computed at runtime
```

---

## Hard Rules

1. **Always ask for the API domain** — never hardcode or assume the URL filter.
2. **Never proceed with a failing test** — run the healer skill first if the test is unstable.
3. **Redact before saving** — no auth tokens or PII in the output files.
4. **Every variable must have a capture rule** — no orphaned `{{placeholders}}`.
5. **Validate idempotency** — blueprints that break data on second run are unusable.
6. **Verify the HAR capture** — always check the file exists and has entries before processing.
7. **Mock files are optional** — only generate when the user explicitly requests mock testing support.
