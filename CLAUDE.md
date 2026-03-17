# Playwright QA Skills

No-code QA automation: record browser workflows and produce stable, production-ready Playwright tests automatically.

This is a **standalone test repository** — it runs separately from the application being tested and points at it via `BASE_URL`.

## Installation

Copy the `skills/` folder into your test repo:

```bash
cp -r skills/ /path/to/your-test-repo/.claude/skills/
```

Then open Claude Code in your test repo and start using the skills via natural language.

## Available Skills

| Say this | Skill triggered | What it does |
|:---|:---|:---|
| "Set up Playwright" | `playwright-project-init` | Scaffolds directories, config, dependencies, and CI pipeline |
| "Set up auth" | `playwright-auth-setup` | Captures login state via `storageState` for all tests |
| "Record the [workflow]" | `playwright-qa-pipeline` | Full pipeline: record → review → refine → heal → done |
| "Review this recording" | `playwright-recording-reviewer` | Scores a recording against a 100-point quality rubric |
| "Refactor this test" | `playwright-test-refiner` | Converts a recording into Page Object Model architecture |
| "Fix this failing test" | `playwright-test-healer` | Diagnoses and repairs using a 6-vector RCA matrix |
| "Capture API traffic" | `playwright-har-recorder` | Records, scrubs, and parameterizes network traffic |

## Folder Structure

```
tests/
  recordings/              ← browser recordings (read-only source)
    reports/               ← quality review scores
  generated/[module]/      ← auto-generated POM tests
    specs/  pages/  fixtures/  utils/  data/
  stable/                  ← tests that passed 3/3 stability gate
  quarantine/              ← tests that couldn't be healed (with RCA notes)
  auth/                    ← storage state files (.gitignored)
  fixtures/                ← shared fixtures
  api-blueprints/          ← HAR blueprints and mock replay files
```

## First Time Setup

1. Copy skills into your test repo's `.claude/skills/`
2. Copy `.env.example` to `.env` and fill in your `BASE_URL` and credentials
3. Say **"Set up Playwright"** to scaffold the project
4. Say **"Set up auth"** if your app requires login
5. Say **"Record the [workflow name]"** to create your first test

## Key Rules

- Recordings are never modified — they are read-only source artifacts
- Tests must pass 3 consecutive times before being promoted to `stable/`
- Auth is handled via `storageState` in fixtures — never inline in tests
- All test interactions go through Page Object Models — no `page.locator()` in spec files
- Tests include cleanup/teardown to avoid polluting the test environment
