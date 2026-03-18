# playwright-qa-skills

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Playwright](https://img.shields.io/badge/Playwright-QA-blue.svg)](https://playwright.dev/)
[![Claude Code](https://img.shields.io/badge/Powered%20by-Claude%20Code-purple.svg)](https://claude.ai/)

> Record once. Get a stable, production-ready Playwright test automatically.

A no-code QA automation solution powered by **Claude Code** skills. Record a browser workflow and let the pipeline handle everything else — project setup, authentication, quality review, refactoring into **Page Object Models**, healing flaky tests, and capturing API blueprints.

**Built for QA teams who want production-grade Playwright tests without writing code.**

---

## What It Does

```
"Set up my project"
           ↓
Init      → Scaffolds directory structure, config, and CI pipeline
           ↓
Auth      → Captures login state so tests skip the login flow
           ↓
"Record the login and search workflow"
           ↓
Record    → Opens browser — you perform the workflow, it records everything
           ↓
Review    → Scores your recording against best practices (100-point rubric)
           ↓
Refine    → Converts recording into Page Object Model architecture
           ↓
Heal      → Runs and fixes the test until it passes 3 times in a row
           ↓
HAR       → (optional) Captures API traffic into parameterized blueprints
           ↓
Stable test ready in tests/stable/
```

---

## Skills Included (7 total)

### Setup & Configuration

| Skill | What it does |
|:---|:---|
| `playwright-project-init` | One-command project setup — directory structure, `playwright.config.ts`, dependencies, CI pipeline |
| `playwright-auth-setup` | Captures login state via `storageState` so tests start already authenticated |

### Test Pipeline

| Skill | What it does |
|:---|:---|
| `playwright-qa-pipeline` | Orchestrator — runs the full pipeline from recording to stable test |
| `playwright-recording-reviewer` | Scores recordings against best practices; blocks low-quality recordings |
| `playwright-test-refiner` | Refactors into POM architecture with fixtures, data factories, and cleanup hooks |
| `playwright-test-healer` | Diagnoses failures using a 6-vector RCA matrix; 3/3 stability gate |
| `playwright-har-recorder` | Captures and parameterizes API traffic into reusable, environment-agnostic blueprints |

---

## First Time Setup

This is a **standalone test repository** — it runs separately from the application being tested and points at it via a `BASE_URL` environment variable.

### Step 1: Copy skills into your test repo

```bash
# From your test repo root
mkdir -p .claude/skills
cp -r /path/to/playwright-qa-skills/skills/* .claude/skills/
```

### Step 2: Set up environment variables

```bash
# Copy the template
cp /path/to/playwright-qa-skills/.env.example .env

# Edit .env and fill in your values
# BASE_URL=https://staging.example.com
# TEST_USER_EMAIL=your-test-user@example.com
# TEST_USER_PASSWORD=your-test-password
```

### Step 3: Scaffold the project

Open Claude Code in your test repo and say:

```
Set up Playwright for testing
```

Claude will create the directory structure, install dependencies, generate `playwright.config.ts`, and optionally set up a CI pipeline.

### Step 4: Set up authentication (if your app requires login)

```
Set up auth for my tests
```

A browser opens — log in normally — close the browser when done. Auth state is saved for all future tests.

### Step 5: Record your first test

```
Record the search and filter workflow
```

Claude will:
1. Ask for the URL, test name, module name, and whether to capture API traffic
2. Open a browser — you perform the workflow
3. Close the browser when finished
4. The pipeline runs automatically: review → refine → heal → done

---

## Output Structure

```
tests/
  recordings/                   ← your original recording (never modified)
    reports/                    ← review scores and feedback
  generated/[module]/
    specs/                      ← refactored test files
    pages/                      ← Page Object Models
    fixtures/                   ← fixture definitions
    utils/                      ← shared helpers and data factories
    data/                       ← test data and constants
  stable/                       ← tests that passed the 3/3 stability gate
  quarantine/                   ← tests that couldn't be healed (with RCA notes)
  auth/                         ← storage state files (.gitignored)
  fixtures/                     ← shared fixtures (auth, HAR)
  api-blueprints/[module]/      ← HAR blueprints, variables, dependency maps, mock replay
```

---

## How Each Skill Triggers

The skills use natural language triggers — just describe what you need:

| What you say | Skill invoked |
|:---|:---|
| "Set up Playwright" / "Initialize the test project" | `playwright-project-init` |
| "Set up auth" / "Configure login for tests" | `playwright-auth-setup` |
| "Record [workflow name]" | `playwright-qa-pipeline` |
| "Review this recording" | `playwright-recording-reviewer` |
| "Refactor this test into POM" | `playwright-test-refiner` |
| "This test is failing, fix it" | `playwright-test-healer` |
| "Capture API traffic from this test" | `playwright-har-recorder` |

---

## Key Features

### No-Code Recording
QA analysts record workflows by interacting with the browser normally. No code knowledge required.

### Quality-Gated Pipeline
Recordings are scored against a 100-point rubric. Low-quality recordings are blocked before they become technical debt.

### Enterprise-Grade Output
Recordings are automatically refactored into Page Object Model architecture with:
- TypeScript interfaces for type safety
- Fixture-based dependency injection
- Data factories for parallel-safe test data
- Cleanup/teardown hooks to prevent environment pollution

### Self-Healing Tests
The healer diagnoses failures using a 6-vector matrix (timing, selectors, test data, interactions, visuals, runtime) and applies targeted fixes — up to 5 iterations.

### 3/3 Stability Gate
A test must pass 3 consecutive times before it's considered stable. One lucky pass is not enough.

### Auth-First Design
Authentication is captured once via `storageState` and shared across all tests. No login flows in test code.

### API Blueprints
Optionally capture, scrub, and parameterize network traffic into environment-agnostic HAR blueprints with mock replay support.

### CI/CD Ready
Project init generates GitHub Actions or GitLab CI pipelines out of the box.

---

## Requirements

- Claude Code with skills support
- Node.js 18+
- `@playwright/test` (installed automatically by `playwright-project-init`)
- A running web application to test

---

## Philosophy

- **No-code first**: Start from real user interactions, not manually written tests
- **Quality-gated**: Low-quality recordings are blocked before they become technical debt
- **Stability-first**: A test that passes once is not stable — 3/3 consecutive passes required
- **Auth-first**: Login is handled once, centrally — never repeated in tests
- **Clean environments**: Every test cleans up after itself
- **Generic**: Works with any web application — no framework-specific dependencies
- **Traceable**: Every artifact links back to its source recording

---

## Contributing

Contributions are welcome! Please read our [Contributing Guide](CONTRIBUTING.md) and [Code of Conduct](CODE_OF_CONDUCT.md) before submitting a pull request.

## Community & Support

- **Issues**: Use [GitHub Issues](https://github.com/sharmasundip/playwright-qa-skills/issues) for bug reports and feature requests.
- **Discussions**: (Optional) Join our [GitHub Discussions](https://github.com/sharmasundip/playwright-qa-skills/discussions) for questions and ideas.
- **Security**: Please report security vulnerabilities according to our [Security Policy](SECURITY.md).

---

## License

MIT — Copyright (c) 2025 playwright-qa-skills contributors. See [LICENSE](LICENSE) for details.
