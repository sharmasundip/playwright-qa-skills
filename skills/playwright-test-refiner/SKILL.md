---
name: playwright-test-refiner
description: Use when a raw Playwright test needs to be refactored into a maintainable Page Object Model architecture with proper encapsulation, semantic locators, and fixture-driven design.
---

# Playwright Test Refiner

You are a Principal SDET. Your mission is to refactor raw Playwright scripts into a fully encapsulated, enterprise-grade architecture using the Page Object Model pattern.

---

## Phase 0: Discovery — Check Before Creating

<IMPORTANT>
Before creating any new files, scan for existing code that can be extended:
</IMPORTANT>

1. **Check `tests/generated/[module]/pages/`** — does a Page Object for this module already exist? If so, extend it rather than creating a duplicate.
2. **Check `tests/generated/[module]/fixtures/`** — is the POM already registered as a fixture?
3. **Check `tests/generated/[module]/utils/`** — are there shared helpers that can be reused?
4. **Check `tests/fixtures/`** — are there global fixtures (auth, HAR) that should be composed with module fixtures?

---

## Architectural Contract

All output files MUST be placed under `tests/generated/[module]/` using this structure:

```
tests/generated/[module]/
  specs/          ← test files (.spec.ts)
  pages/          ← Page Object Models
  fixtures/       ← fixture definitions
  utils/          ← shared helpers and state persistence
  data/           ← static test data and constants
```

Where `[module]` is the feature area provided by the user (e.g., `search`, `settings`, `auth`).

---

## The Entity-State Contract

Every Page Object MUST follow this contract:

1. **TypeScript interface** — define an interface for the entity at the top of the page file:
   ```typescript
   export interface RecordEntity {
     id: string;
     name: string;
     status: string;
   }
   ```

2. **State return** — `create()` methods must return `Promise<EntityInterface>`:
   ```typescript
   async create(data: RecordData): Promise<RecordEntity> {
     // ... interactions
     return { id, name, status };
   }
   ```

3. **Entity ID capture strategy** (in priority order):
   - Primary: API response interception (capture from network response)
   - Secondary: URL regex match after navigation
   - Tertiary: DOM scrape from a confirmation element

4. **Self-validation** — POMs must include a `validate()` method:
   ```typescript
   async validate(entity: RecordEntity): Promise<void> {
     await expect(this.page.getByTestId('record-name'))
       .toHaveText(entity.name);
   }
   ```

5. **Cleanup method** — POMs should include a `cleanup()` or `delete()` method when the entity supports deletion:
   ```typescript
   async delete(entity: RecordEntity): Promise<void> {
     await this.page.goto(`/records/${entity.id}`);
     await this.page.getByRole('button', { name: 'Delete' }).click();
     await this.page.getByRole('button', { name: 'Confirm' }).click();
     await expect(this.page.getByText('Record deleted')).toBeVisible();
   }
   ```

   If the entity cannot be deleted via UI, add a comment:
   ```typescript
   // TODO: No UI delete available. Consider API cleanup in test teardown.
   ```

---

## Locator Priority and Forbidden Patterns

**Use locators in this priority order:**

| Priority | Locator | Example | When to Use |
|:---|:---|:---|:---|
| 1st | `getByTestId` | `page.getByTestId('submit-btn')` | When `data-testid` exists |
| 2nd | `getByRole` | `page.getByRole('button', { name: 'Save' })` | For interactive elements |
| 3rd | `getByLabel` | `page.getByLabel('Email address')` | For form inputs |
| 4th | `getByPlaceholder` | `page.getByPlaceholder('Search...')` | For inputs with placeholder |
| 5th | `getByText` | `page.getByText('Confirm action')` | For static text elements |

**Forbidden locators:**

- `page.locator('xpath=...')` — never use xpath
- `page.locator('.class-name')` — never use CSS class selectors
- `page.locator(':nth-child(3)')` — never use positional nth selectors
- `page.locator('#id')` — only use if no semantic alternative exists

<IMPORTANT>
If a forbidden locator is unavoidable (e.g., no testId attribute exists on an element):
1. Add the comment: `// TODO: Replace forbidden locator once a testId is added to this element`
2. STOP the migration for this file
3. Flag it as needing review — do not silently continue
</IMPORTANT>

---

## Multi-Page Flow Pattern

For workflows that span multiple pages (e.g., a multi-step wizard or form), create one POM per page and compose them in the fixture:

```typescript
// tests/generated/[module]/pages/StepOnePage.ts
export class StepOnePage {
  constructor(private page: Page) {}

  async fillDetails(data: StepOneData): Promise<void> { /* ... */ }
  async proceedToStepTwo(): Promise<void> {
    await this.page.getByRole('button', { name: 'Continue' }).click();
    await this.page.waitForURL('**/step-two');
  }
}

// tests/generated/[module]/pages/StepTwoPage.ts
export class StepTwoPage {
  constructor(private page: Page) {}

  async fillDetails(data: StepTwoData): Promise<void> { /* ... */ }
  async proceedToStepThree(): Promise<void> {
    await this.page.getByRole('button', { name: 'Continue' }).click();
    await this.page.waitForURL('**/step-three');
  }
}

// tests/generated/[module]/fixtures/[module].fixture.ts
export const test = base.extend<{
  stepOnePage: StepOnePage;
  stepTwoPage: StepTwoPage;
  stepThreePage: StepThreePage;
  confirmationPage: ConfirmationPage;
}>({
  stepOnePage: async ({ page }, use) => { await use(new StepOnePage(page)); },
  stepTwoPage: async ({ page }, use) => { await use(new StepTwoPage(page)); },
  stepThreePage: async ({ page }, use) => { await use(new StepThreePage(page)); },
  confirmationPage: async ({ page }, use) => { await use(new ConfirmationPage(page)); },
});
```

---

## Test Data Management

### Static Data (known values)

Place in `tests/generated/[module]/data/[module].data.ts`:

```typescript
export const sampleData = {
  valid: {
    name: 'Sample Record A',
    description: 'A valid test record',
    category: 'general',
  },
  minimal: {
    name: 'Minimal Record',
  },
} as const;
```

### Dynamic Data (generated per run)

Use a helper in `tests/generated/[module]/utils/data-factory.ts`:

```typescript
export function generateUniqueRecord(): RecordData {
  const timestamp = Date.now();
  return {
    name: `Test Record ${timestamp}`,
    description: 'Auto-generated test data',
    reference: `REF-${timestamp}`,
  };
}
```

This prevents test collisions when running in parallel or repeatedly.

---

## Refactoring Protocol

### Step 1: Analyze the raw test

Read the raw spec and identify:
- The user flow (what actions are performed in sequence)
- How many pages are involved (single page vs. multi-page flow)
- All page interactions that should become POM methods
- All assertions that should become POM validators
- Any hardcoded data that should move to `data/`
- Auth/session setup (should use `storageState` or fixture)
- Whether cleanup/teardown is needed

### Step 2: Write or update the Page Object

File: `tests/generated/[module]/pages/[Module]Page.ts`

```typescript
import { Page, expect } from '@playwright/test';

export interface RecordEntity {
  id: string;
  name: string;
  category: string;
}

export class RecordPage {
  constructor(private page: Page) {}

  async navigate(): Promise<void> {
    await this.page.goto('/records');
    await this.page.waitForLoadState('networkidle');
  }

  async create(data: { name: string; category: string }): Promise<RecordEntity> {
    await this.page.getByRole('button', { name: 'Add New' }).click();
    await this.page.getByLabel('Name').fill(data.name);
    await this.page.getByLabel('Category').fill(data.category);

    const [response] = await Promise.all([
      this.page.waitForResponse(res => res.url().includes('/api/records') && res.status() === 201),
      this.page.getByRole('button', { name: 'Save' }).click(),
    ]);

    const body = await response.json();
    return { id: body.data.id, name: data.name, category: data.category };
  }

  async validate(entity: RecordEntity): Promise<void> {
    await expect(this.page.getByTestId('record-name')).toHaveText(entity.name);
  }

  async delete(entity: RecordEntity): Promise<void> {
    await this.page.goto(`/records/${entity.id}`);
    await this.page.getByRole('button', { name: 'Delete' }).click();
    await this.page.getByRole('button', { name: 'Confirm' }).click();
    await expect(this.page.getByText('Record deleted')).toBeVisible();
  }
}
```

### Step 3: Register in fixture

File: `tests/generated/[module]/fixtures/[module].fixture.ts`

```typescript
import { test as base } from '@playwright/test';
import { RecordPage } from '../pages/RecordPage';

type Fixtures = {
  recordPage: RecordPage;
};

export const test = base.extend<Fixtures>({
  recordPage: async ({ page }, use) => {
    await use(new RecordPage(page));
  },
});

export { expect } from '@playwright/test';
```

### Step 4: Write the spec with teardown

File: `tests/generated/[module]/specs/[test-name].spec.ts`

```typescript
import { test, expect } from '../fixtures/[module].fixture';

test.describe('Record creation', () => {
  let createdRecord: RecordEntity;

  test('should create a record and verify it appears in the list', async ({ recordPage }) => {
    await recordPage.navigate();

    createdRecord = await recordPage.create({ name: 'Sample A', category: 'General' });
    await recordPage.validate(createdRecord);
  });

  test.afterEach(async ({ recordPage }) => {
    // Clean up created test data to avoid environment pollution
    if (createdRecord?.id) {
      await recordPage.delete(createdRecord).catch(() => {
        // Cleanup failure is non-fatal — log but don't fail the test
      });
    }
  });
});
```

**The spec file must have zero `page.locator()` calls** — all interactions go through the POM.

---

## Pre-Flight Checklist

Before declaring the refactor complete, verify:

- [ ] All test files are under `tests/generated/[module]/`
- [ ] Spec file has zero direct `page.locator()` calls
- [ ] POM has a TypeScript interface for the entity
- [ ] `create()` returns `Promise<EntityInterface>`
- [ ] POM has a `validate()` method
- [ ] POM has a `delete()` or `cleanup()` method (or a TODO comment explaining why not)
- [ ] No forbidden locators (or all are flagged with `// TODO` comment)
- [ ] Fixture properly injects the POM into the spec
- [ ] Hardcoded test data moved to `data/` or `utils/data-factory.ts`
- [ ] Test has teardown/cleanup in `afterEach` or `afterAll`

---

## Hard Rules

1. Never put test logic directly in spec files — all interactions go in the POM.
2. Never use xpath or CSS class selectors without a `// TODO` comment and a halt.
3. Never duplicate a POM — check for existing ones and extend them.
4. Fixture injection only — no direct class instantiation in specs.
5. Auth goes in fixture setup via `storageState`, never inline in the test.
6. Always include cleanup/teardown — tests that leave behind data are a liability.
7. Use data factories for unique values — hardcoded strings cause parallel test collisions.
