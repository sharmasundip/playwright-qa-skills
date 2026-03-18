# AI Agent Instructions (AIPROMPT)

This repository is optimized for autonomous AI agents (like Claude Code). When working in this repository, follow these guidelines to ensure consistency and high-quality output.

## Repository Purpose
A collection of Claude Code skills that automate the creation and maintenance of Playwright tests using a "No-Code" capture-first approach.

## Key Architectures
- **Skills**: Located in `skills/`. Each subdirectory contains a `SKILL.md` defining its behavior.
- **Pipeline**:
  1. `playwright-recording-reviewer`: Validates the quality of a raw recording.
  2. `playwright-test-refiner`: Refactors raw code into Page Object Models (POM).
  3. `playwright-test-healer`: Iteratively fixes tests until they pass.
- **Data Flow**: `tests/recordings/` (Source) -> `tests/generated/` (Transformation) -> `tests/stable/` (Final).

## Guidelines for Agents
1. **Skill Invocation**: Use natural language triggers defined in `CLAUDE.md`.
2. **Stability Gate**: A test is not "done" until it passes 3 consecutive times in the `stable/` directory.
3. **POM Standards**: Use TypeScript interfaces for all Page Objects. Group locators at the top of the class.
4. **Error Handling**: If `playwright-test-healer` fails after 5 iterations, move the test to `tests/quarantine/` with a detailed RCA note.

## Common Workflows
- **New Test**: `Record the [workflow]` -> Let the pipeline run.
- **Fixing Flaky Tests**: `Fix this failing test [path]` -> Healer skill.
- **Updating Auth**: `Set up auth` -> Captures new `storageState.json`.
