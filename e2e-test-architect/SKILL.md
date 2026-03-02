---
name: e2e-test-architect
description: "Apply best practices to generate, run, and document E2E (end-to-end) tests for frontend projects. Automatically detects the testing framework (Playwright or Cypress), reads real source code before writing tests, organizes test files by feature/page, executes tests in terminal with a self-healing loop, and maintains an E2E_TRACKER.md document. Trigger when user mentions: E2E, playwright, cypress, 端到端测试, 自动化测试, e2e test, end-to-end test, write E2E tests, run E2E tests, or asks to test a user flow / user journey."
---

# E2E Test Architect

Automated workflow for writing, executing, and documenting production-grade E2E tests.

<workflow>

## Step 1 — Environment Check

<step_1_environment_check>

Before writing any test code:

1. Read `package.json` at the project root.
2. Inspect `dependencies` and `devDependencies` to detect the installed E2E framework:
   - `@playwright/test` → Playwright
   - `cypress` → Cypress
   - If neither is found, ask the user which framework to install. Default recommendation: Playwright.
3. Identify the test runner command from `scripts` (e.g., `"test:e2e"`, `"e2e"`, `"playwright"`, `"cypress"`).
4. Detect TypeScript support (`typescript` in deps, or `tsconfig.json` exists).
   - **Always default to TypeScript.** Use `.spec.ts` for Playwright, `.cy.ts` for Cypress.
   - Only fall back to `.js` if the project explicitly has no TypeScript setup AND the user confirms.
5. Read any existing E2E config file (`playwright.config.ts`, `cypress.config.ts`) to understand:
   - `baseURL` / base path
   - Custom test directory
   - Browser targets
   - Viewport settings
   - Timeout values

</step_1_environment_check>

## Step 2 — Investigate Before Writing

<step_2_investigate>

**Never guess DOM structure. Never fabricate selectors.**

Before writing any test for a page or feature:

1. Identify the target page/component file(s) by reading the routing configuration
   (e.g., `src/router/`, `src/app/`, `pages/`, `next.config.*`).
2. Read the **actual source code** of every component involved in the user flow.
3. Read related state management files (stores, contexts, reducers) to understand data flow.
4. Read API call modules to understand request/response shapes (useful for mocking or intercepting).
5. From the source code, extract:
   - Existing `data-testid` attributes.
   - User-visible text strings (button labels, headings, placeholders).
   - ARIA roles and labels.
   - Form field names and types.
   - Navigation paths and route parameters.
6. Document the user journey as a numbered list of interactions before writing any code.

</step_2_investigate>

## Step 3 — File Management

<step_3_file_management>

Organize test files by feature or page. Never dump all tests into a flat directory.

### Directory Structure

```
{e2e-root}/            # e.g., tests/e2e/ or e2e/ — match project config
├── auth/
│   ├── login.spec.ts
│   ├── register.spec.ts
│   └── password-reset.spec.ts
├── checkout/
│   ├── cart.spec.ts
│   └── payment.spec.ts
├── dashboard/
│   └── overview.spec.ts
├── fixtures/          # Shared test data
│   └── users.ts
├── helpers/           # Shared utilities (custom commands, page objects)
│   └── navigation.ts
└── E2E_TRACKER.md     # Auto-maintained documentation
```

### Rules

- Determine the E2E root directory from the framework config. If none is configured, use `e2e/`.
- Create a sub-folder for each feature domain or page group.
- Name test files after the specific flow they cover (e.g., `login.spec.ts`, not `test1.spec.ts`).
- Extract shared setup, fixtures, and helpers into dedicated directories.
- Keep each test file focused on one feature area. Aim for fewer than 200 lines per file.

</step_3_file_management>

## Step 4 — Execution Loop

<step_4_execution_loop>

After writing tests to files, execute them immediately. Never claim completion without a passing run.

### Playwright

```bash
npx playwright test {path-to-spec-file} --reporter=list
```

### Cypress

```bash
npx cypress run --spec "{path-to-spec-file}"
```

### Self-Healing Loop

1. Run the test command.
2. If the test **passes** → proceed to Step 5.
3. If the test **fails**:
   a. Read the full terminal error output carefully.
   b. Identify the root cause (wrong selector, timing issue, missing mock, incorrect assertion).
   c. If the failure suggests a selector mismatch, re-read the source component to find the correct selector.
   d. Fix the test code and save the file.
   e. Re-run the test.
   f. Repeat until all tests pass.
4. Maximum retry attempts: 5. If still failing after 5 attempts, report the failure details to the user
   with a diagnosis and ask for guidance.

### Debugging Tips

- For Playwright: use `--debug` flag or `page.pause()` for interactive debugging when stuck.
- For Cypress: check screenshots/videos in `cypress/screenshots/` and `cypress/videos/`.
- Check for race conditions: add explicit waits for network requests or DOM elements rather than arbitrary timeouts.

</step_4_execution_loop>

## Step 5 — Update Documentation

<step_5_update_documentation>

After tests pass, create or update `{e2e-root}/E2E_TRACKER.md`.

### Tracker Format

```markdown
# E2E Test Tracker

> Auto-generated by e2e-test-architect. Last updated: YYYY-MM-DD HH:mm

## Coverage Summary

| Feature       | Test File                     | User Journeys Covered         | Status |
|---------------|-------------------------------|-------------------------------|--------|
| Authentication| `auth/login.spec.ts`          | Login with valid credentials, Login error handling | ✅ Pass |
| Authentication| `auth/register.spec.ts`       | New user registration flow    | ✅ Pass |
| Checkout      | `checkout/payment.spec.ts`    | Complete purchase flow        | ✅ Pass |

## Recent Changes

- **YYYY-MM-DD** — Added `auth/login.spec.ts`: covers login success and error paths.
- **YYYY-MM-DD** — Updated `checkout/payment.spec.ts`: added coupon code scenario.
```

### Rules

- If `E2E_TRACKER.md` already exists, read it first, then append or update the relevant rows.
- Always update the "Last updated" timestamp.
- Add a new entry under "Recent Changes" for every create/modify action.

</step_5_update_documentation>

</workflow>

<strict_rules>

## Strict Rules

1. **TypeScript only.** All test files must use `.spec.ts` (Playwright) or `.cy.ts` (Cypress).
2. **No large code blocks in chat.** Write test code directly to files using file-writing tools. Only show short snippets (fewer than 15 lines) in chat when explaining a concept.
3. **No premature completion.** Never say "done" or "tests are ready" until the test has actually passed in the terminal. The execution loop (Step 4) is mandatory.
4. **Robust locators only.** Prefer locators in this priority order:
   - `data-testid` attributes
   - ARIA roles and labels (`getByRole`, `getByLabel`)
   - User-visible text (`getByText`, `getByPlaceholder`)
   - Semantic HTML elements (`button`, `input[type="email"]`)
   - **Last resort only:** CSS class selectors or XPath. If forced to use them, add a comment explaining why.
5. **No hardcoded waits.** Never use `page.waitForTimeout(ms)` or `cy.wait(ms)` with arbitrary durations. Use explicit conditions: `waitForSelector`, `waitForResponse`, `waitForURL`, or Playwright auto-waiting.
6. **Isolate tests.** Each `test()` / `it()` block must be independent. No test should depend on the side effects of another test. Use `beforeEach` for shared setup.
7. **Clean assertions.** Use framework-native assertions (`expect` in Playwright, `should`/`expect` in Cypress). Never use raw `if/else` for validation logic.
8. **Never trust feature code.** Do NOT modify test files just to make tests pass. If the test logic is confirmed correct and reasonable but still fails, suspect the feature code first. Tests are the source of truth for expected behavior.
9. **Never fix feature code.** This skill is responsible for writing and running E2E tests only, NOT for fixing application bugs. If a feature has a bug that causes a test to fail, complete all remaining tests first, then investigate the root cause and output a bug report document to `{e2e-root}/BUG_REPORT.md` — do NOT apply fixes to the feature code.
10. **Audit existing test files.** If the project already has E2E test files, read and review them before writing new ones. Check whether they are correct, up-to-date, and aligned with the current source code (selectors, routes, API shapes). If issues are found, output a review document to `{e2e-root}/TEST_REVIEW.md` describing each problem.
11. **Output language and location.** All generated documentation files (E2E_TRACKER.md, BUG_REPORT.md, TEST_REVIEW.md, etc.) must be placed inside the E2E root directory. The document language must match the user's input language; fall back to English if the user's language cannot be determined.

</strict_rules>
