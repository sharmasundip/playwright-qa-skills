---
name: playwright-qa-pipeline
description: Use when the user wants to record a browser workflow and automatically produce a stable, production-ready Playwright test — handling review, refactoring, healing, and optional API blueprint generation in one session.
---

# Playwright QA Pipeline

A single-command pipeline: record a browser workflow, then automatically review, refine, heal, and optionally capture API blueprints — no manual steps required.

---

## Step 0: Environment Pre-Check

Before anything else, verify the project is ready:

### 0.1: Check Project Setup

```bash
npx playwright --version 2>/dev/null
```

| Result | Action |
|:---|:---|
| Version prints | Playwright is installed — continue |
| Command not found | Invoke the `playwright-project-init` skill first, then resume this pipeline |

### 0.2: Check Auth State (if app requires login)

```bash
test -s tests/auth/storage-state.json && echo "AUTH_READY" || echo "AUTH_MISSING"
```

| Result | Action |
|:---|:---|
| `AUTH_READY` | Continue |
| `AUTH_MISSING` | Ask user: "Does this app require login?" If yes → invoke `playwright-auth-setup` skill first |

### 0.3: Check Environment Variables

```bash
test -f .env && echo "ENV_EXISTS" || echo "ENV_MISSING"
```

If `.env` is missing but `playwright.config.ts` references `process.env.BASE_URL`, warn the user:

> "Your config references environment variables but no `.env` file exists. Create one from `.env.example` or provide the BASE_URL directly."

---

## Step 1: Gather Inputs

Before opening the browser, ask the user for:

1. **URL** — what page to open for recording (e.g., `https://staging.example.com/home`)
2. **Test name** — a short kebab-case name (e.g., `create-record`, `search-by-name`)
3. **Module name** — the feature area this test belongs to (e.g., `search`, `settings`, `auth`)
4. **HAR recording** — should API traffic be captured after healing? (yes / no)

Save these as: `RAW_FILE=tests/recordings/[test-name].spec.ts`, `MODULE=[module-name]`.

**Validation:**
- Test name must be kebab-case (lowercase, hyphens only): `create-record` not `CreateRecord`
- Check if `tests/recordings/[test-name].spec.ts` already exists — if so, ask if user wants to overwrite or choose a new name
- Module name should be lowercase, single word or hyphenated: `search`, `user-management`

---

## Step 2: Start Recording Session

Open the browser recorder using Playwright codegen:

```bash
npx playwright codegen [URL] --output tests/recordings/[test-name].spec.ts
```

If the app requires login and the user wants to re-capture auth at the same time:

```bash
npx playwright codegen [URL] --save-storage=tests/auth/storage-state.json --output tests/recordings/[test-name].spec.ts
```

If auth is already captured and the user wants to start the recording pre-authenticated:

```bash
npx playwright codegen --load-storage=tests/auth/storage-state.json [URL] --output tests/recordings/[test-name].spec.ts
```

Tell the user:

> "The browser is open with the Playwright recorder. Perform your workflow step by step — every click, fill, and navigation is being captured automatically. When you are done, close the browser window and I will process the recorded test."

**Recording tips to share with the user:**
- Click elements naturally — the recorder will choose the best locator automatically
- If you need to assert something, use the **assertion toolbar** at the top of the inspector (the checkmark icon) to add `expect()` checks
- Avoid hovering over elements unnecessarily — it may record unwanted hover actions
- If you make a mistake, keep going — we will clean it up in the refine step
- For dropdowns: click to open, then click the option — don't try to use keyboard navigation

While the user records:
- Let the user drive all interactions — do NOT click or fill anything yourself

When the user closes the browser:
- Verify the file was saved to `tests/recordings/[test-name].spec.ts`
- Read the generated file and confirm it contains test actions
- If the file is empty or missing, ask the user to re-record

---

## Step 3: Quality Review (Automatic)

Invoke the `playwright-recording-reviewer` skill on the newly saved file.

The reviewer will score the recording out of 100 and assign a grade:

| Grade | Score | Action |
|:---|:---:|:---|
| A or B | 80–100 | Proceed automatically |
| C | 70–79 | Proceed with a warning noting issues to fix later |
| D or F | < 70 | **STOP** — offer re-recording |

<IMPORTANT>
If score is below 70 (grade D or F):

1. Stop the pipeline
2. Show the reviewer's issues report to the user in a clear, non-technical format
3. Offer two options:
   - **Re-record**: "Would you like to re-record the workflow? I'll open the browser again."
   - **Manual fix**: "I can try to fix the common issues automatically and re-score. Want me to try?"
4. If the user chooses re-record, go back to Step 2
5. If the user chooses manual fix, attempt auto-corrections (add missing assertions, improve locators) and re-score
6. Do NOT proceed to refactoring with a failing-grade recording
</IMPORTANT>

---

## Step 4: Refine (Automatic)

Invoke the `playwright-test-refiner` skill on `tests/recordings/[test-name].spec.ts`.

The refiner will:
- Convert the raw test into a Page Object Model architecture
- Output to `tests/generated/[module]/specs/[test-name].spec.ts`
- Create companion POM, fixture, and utility files

If the refiner halts due to a forbidden locator issue:
1. Report it to the user in plain language: "The app uses a CSS class `.btn-primary` for the submit button. This could break if the design changes."
2. Offer: "I can add a `// TODO` comment and continue, or you can ask your developer to add a `data-testid` attribute."
3. If user says continue, proceed with TODO comment

---

## Step 5: Heal (Automatic)

Invoke the `playwright-test-healer` skill on the refined test.

The healer will:
- Run the test and diagnose any failures using the 6-vector diagnostic matrix
- Apply up to 5 fix iterations
- Verify stability with the 3/3 pass rule (must pass 3 consecutive times)

Outcomes:
- **Stable** → test moved to `tests/stable/[test-name].spec.ts`
- **Unresolved after 5 iterations** → test moved to `tests/quarantine/[test-name].spec.ts` with `test.fixme()` and RCA findings in comments

**Progress updates during healing:**

After each iteration, briefly tell the user what happened:

> "Iteration 2/5: Fixed a timing issue — the page wasn't fully loaded before clicking. Running again..."

<IMPORTANT>
Never mark a test as stable unless it has passed 3 consecutive runs. One lucky pass is not sufficient.
</IMPORTANT>

---

## Step 6: HAR Recording (If Requested)

If the user requested HAR recording in Step 1:

Invoke the `playwright-har-recorder` skill on the validated test.

The recorder will:
- Ask the user: "What API domain or URL pattern should be captured?" (e.g., `api.example.com` or `/api/v1/`)
- Capture, scrub, and parameterize the network traffic
- Output blueprints to `tests/api-blueprints/[module]/[test-name].har.json`

---

## Step 7: Final Outcome Report

Always end the pipeline with a clear summary:

```
Pipeline Complete
─────────────────────────────────────────
Test name:        [test-name]
Module:           [module]
Quality score:    [score]/100 ([grade])
Refactored to:    tests/generated/[module]/specs/[test-name].spec.ts
Final location:   tests/stable/[test-name].spec.ts   ← if stable
                  tests/quarantine/[test-name].spec.ts ← if unresolved
Healing rounds:   [N] iteration(s) needed
HAR blueprint:    [yes → tests/api-blueprints/...] / [skipped]
─────────────────────────────────────────
```

### If Pipeline Stopped Early

Always produce a report even on failure:

```
Pipeline Stopped
─────────────────────────────────────────
Test name:        [test-name]
Stopped at:       [Step N: step name]
Reason:           [why it stopped]
Files created:    [list any files that were created]
─────────────────────────────────────────
Next step: [specific action the user should take]
```

### Next Steps Suggestions

After the report, suggest what the user can do next:

- "Run `npx playwright test tests/stable/[test-name].spec.ts --headed` to watch the test run"
- "Record another workflow by telling me what to test next"
- "Run all validated tests with `npx playwright test tests/stable/`"

---

## Hard Rules

1. Never skip the environment pre-check — missing dependencies waste the user's time.
2. Never skip the quality review step — it catches issues before they compound.
3. Never proceed to healing without a successful refactor.
4. Never declare a test stable without 3 consecutive passes.
5. Always produce the outcome report, even if the pipeline stopped early.
6. Never modify the raw recording file — it is a read-only source artifact.
7. Always offer a re-recording option when quality score is below 70.
8. Keep user communications non-technical — this is a no-code solution for QA teams.
