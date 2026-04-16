# saucedemo-e2e — Test Planning Rules

---

## Project Directory Layout

| What | Path |
|------|------|
| **App source code** (components, routing, `data-test` attrs) | `C:\Users\rames\Desktop\Workspace\saucedemo\sample-app-web\src\` |
| **Test harness** (Playwright config, page objects, specs) | `C:\Users\rames\Desktop\Workspace\saucedemo\sample-app-web\tests\` |

When running the setup agent Phase 1 (codebase analysis), always scan the **app source** path above — not a relative `../src/` path, which may silently resolve incorrectly depending on the agent's working directory.

---

## Page Object Model (POM) Convention

**Every feature under test must have a dedicated page object file in `pages/<feature>.page.ts`.** Never write locators or interactions inline in spec files.

### Canonical structure

```typescript
import { Page, Locator } from "@playwright/test";

export default class ExamplePage {
  private readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async goto() {
    await this.page.goto("/example.html");
  }

  // Static locators — getter method returning Locator
  getTitle(): Locator {
    return this.page.locator('[data-test="title"]');
  }

  getSubmitButton(): Locator {
    return this.page.locator('[data-test="submit"]');
  }

  // Dynamic/parameterised locators — method with arguments
  getRemoveButton(slug: string): Locator {
    return this.page.locator(`[data-test="remove-${slug}"]`);
  }

  // Action methods — call getter methods internally
  async submit() {
    await this.getSubmitButton().click();
  }
}
```

### Rules

| Rule | Why |
|------|-----|
| The **only** class-level field is `private readonly page: Page` | Locators created at construction time become stale after navigation; getter methods create fresh locators on demand |
| All locators are exposed via **`get*()`** methods that return `Locator` | Tests assert against returned locators; action methods call getters internally — one source of truth per selector |
| **No** `readonly locatorName: Locator` class fields | These are the stale-locator anti-pattern — do not use |
| **No** bare `this.page.locator(...)` calls inside action methods | Inline selectors bypass the getter layer; if the selector changes it must be updated in multiple places |
| Every feature page has a file in `pages/<name>.page.ts` | Spec files must import and use page objects — locators do not belong in spec files |

### Forbidden anti-patterns

```typescript
// ❌ Anti-pattern 1 — readonly locator fields (stale after navigation)
export default class BadPage {
  private readonly page: Page;
  readonly title: Locator;          // ← WRONG
  constructor(page: Page) {
    this.page = page;
    this.title = page.locator('[data-test="title"]');  // ← WRONG
  }
}

// ❌ Anti-pattern 2 — inline locators inside action methods (no getter layer)
async clickSubmit() {
  await this.page.locator('[data-test="submit"]').click();  // ← WRONG, use getSubmitButton()
}

// ❌ Anti-pattern 3 — locators written directly in spec files (no page object)
test('example', async ({ page }) => {
  await page.locator('[data-test="title"]').isVisible();  // ← WRONG, use a page object
});
```

---

## Mandatory Test Generation Pipeline

**Never use the orchestrator to auto-generate tests for a feature that has never been set up.**
The orchestrator is for re-running known pipelines. For new feature coverage, follow these four stages in order, stopping at each human checkpoint.

```
Stage 1: Setup     →  Stage 2: Review setup  →  Stage 3: Plan  →  Stage 4: Review plan  →  Stage 5: Generate
         (agent)             (human)                      (agent)             (human)                   (agent)
```

---

### Stage 1 — Run the setup agent

Invoke the `playwright-test-setup` agent. It **must** complete both phases:

| Phase | Source | What it catches that the other misses |
|-------|--------|---------------------------------------|
| **Phase 1 — Codebase** | `../src/`, `pages/`, `fixtures/`, `utils/` | All `data-test` attributes (including ones only visible in edge states), conditional rendering logic, localStorage keys, auth guard logic, all possible error strings, role-gated component differences |
| **Phase 2 — Live site** | Playwright MCP browser tools against `BASE_URL` | Exact rendered text (JSX whitespace collapses), what invalid URL params actually render, visual empty states, real navigation behavior, network requests fired, console errors |

Both phases are mandatory. A setup file produced from code alone will miss rendering details. A setup file produced from the live site alone will miss selectors that only appear in edge states.

The setup file is saved to `test-setup/<feature>.setup.md` and must include a **UI Element Coverage Matrix** for every page in the feature:

```
## UI Element Coverage Matrix

### /cart.html — "Your Cart"
| Element | data-test selector | States / Notes |
|---|---|---|
| Page title | [data-test="title"] | Always "Your Cart" |
| Cart item rows | [data-test="inventory-item"] | One per item; absent when cart empty |
| QTY per item | [data-test="item-quantity"] | Always "1" (hardcoded) |
| Remove button | [data-test="remove-{slug}"] | One per item |
| Continue Shopping | [data-test="continue-shopping"] | Always visible |
| Checkout button | [data-test="checkout"] | Always visible |
| Empty state | (no special element) | Headers remain; no items; no message |
```

---

### Stage 2 — Human reviews the setup file

Before proceeding to planning, open `test-setup/<feature>.setup.md` and verify:

- [ ] Every page in the feature has a UI Element Coverage Matrix entry
- [ ] Every interactive element (button, input, link) is listed with its selector
- [ ] Every error message is listed with its exact text
- [ ] Every navigation path (including direct URL access, back button, cancel) is documented
- [ ] Broken/edge states (invalid IDs, empty states, problem_user degradations) are documented
- [ ] Page titles (`[data-test="title"]`) are listed for every route

**Do not proceed to Stage 3 until this checklist is complete.**

---

### Stage 3 — Run the planner agent

The planner reads the setup file and produces `test-plans/<feature>.testplan.md`.

Use the orchestrator with `--plan-only` flag to stop after plan generation:
```
orchestrator: <feature> --plan-only
```

Or invoke the planner agent directly.

---

### Stage 4 — Human reviews the test plan

Cross-check the plan against the UI Element Coverage Matrix from the setup file:

- [ ] Every element in the matrix appears in at least one scenario's steps or expected results
- [ ] Every error message in the matrix has a scenario that asserts its exact text
- [ ] Every navigation path in the matrix has a scenario that exercises it
- [ ] Every broken/edge state in the matrix has a scenario covering it
- [ ] The negative + edge scenario count is ≥ 30% of total scenarios

**Do not proceed to Stage 5 until every matrix row maps to at least one scenario.**

---

### Stage 5 — Generate tests

Only after Stage 4 sign-off, run the generator:
```
orchestrator: <feature> --skip-setup
```

This skips the staleness check and uses the reviewed setup file and plan as-is.

---

## CRITICAL: Every Test Plan Must Cover All Three Case Types

For **every feature**, the test plan MUST explicitly include all three types of scenarios. This is non-negotiable — do not write a plan that only covers happy paths.

| Case Type | Definition | Minimum required |
|-----------|-----------|-----------------|
| **Positive** | User does the right thing and the system responds correctly | One per distinct user flow |
| **Negative** | User hits an error, limit, or permission wall — system blocks and communicates clearly | One per error condition, limit, and access rule |
| **Edge Case** | Boundary conditions, unusual but valid states, unexpected sequences | One per identified boundary or state transition |

### Before writing any scenario, ask these three questions for every flow:
1. **Positive** — What does success look like? Does every entry point to this action succeed?
2. **Negative** — What blocks this action? (permission denied, limit reached, resource not found, network failure, invalid input) — write a test for each blocker.
3. **Edge Case** — What unusual-but-valid states exist? (empty state, at exact limit, dismiss without completing, reload after action, action during pending state)

If a test plan contains fewer than 30% negative + edge case scenarios, it is incomplete.

---

## Mandatory Test Plan Coverage Checklist

When generating a test plan for **any** feature, the planner MUST produce at least one scenario for every applicable category below. Do not skip a category without explicitly stating why it does not apply.

---

### 1. Role Coverage

SauceDemo has three test projects with distinct user contexts:

| Project | SauceDemo user | Spec suffix | When to use |
|---------|---------------|-------------|-------------|
| `no-auth` | — (not logged in) | `*.noauth.spec.ts` | Login page tests, locked-out user, redirect-to-login |
| `standard` | `standard_user` | `*.standard.spec.ts` | Primary happy-path and feature coverage |
| `problem` | `problem_user` | `*.problem.spec.ts` | Degraded UI, broken images, error conditions |

For every role that has different behaviour or UI:
- [ ] NoAuth — is the user redirected to the login page?
- [ ] Standard — can they perform the action end-to-end?
- [ ] Problem — does the UI degrade gracefully or show expected errors?

---

### 2. Happy Path (one per distinct user flow)
- [ ] Primary action succeeds end-to-end (add / create / submit)
- [ ] Secondary action succeeds (edit / update / toggle)
- [ ] Removal/delete action succeeds

---

### 3. Empty / Zero State
- [ ] What does the UI show when there is no data yet (0 items, 0 reactions, empty list)?

---

### 4. Boundary / Limit Conditions
For any numeric limit (max reactions, max list name length, max items, etc.):
- [ ] At limit minus 1 (N−1): action still succeeds
- [ ] At limit exactly (N): action succeeds, UI reflects limit reached
- [ ] Over limit (N+1): action is blocked, correct toast/error shown

---

### 5. All Paths to the Same Action
If the same action can be triggered from multiple UI entry points (e.g. a button on a list card AND a button on a detail page):
- [ ] Test each entry point independently

---

### 6. Abandonment / Cancel / Dismiss
For every modal, popover, or picker:
- [ ] Open and dismiss without completing the action (click outside, press Escape, scroll away)
- [ ] State is unchanged after dismissal
- [ ] Component can be re-opened after dismissal

---

### 7. Optimistic Update Rollback (Network / Server Error)
For every mutation (PUT, POST, DELETE) that has an optimistic UI update:
- [ ] Intercept the API call and return a 5xx error
- [ ] Verify the optimistic change is reverted
- [ ] Verify an error toast is shown

---

### 8. Persistence After Reload
For any action that writes to the server:
- [ ] Perform the action, reload the page, verify the change survived

---

### 9. Async / Pending State
For any mutation that takes >0ms:
- [ ] While the mutation is in-flight, interactive elements (buttons, pills) are disabled/pending
- [ ] Elements become interactive again after the mutation settles

---

### 10. Navigation Edge Cases
For any page that takes a resource ID from the URL:
- [ ] Non-UUID / malformed ID in URL → graceful error, no crash
- [ ] Valid UUID that does not exist in DB → graceful 404 state, no crash

---

### 11. Filtering / Conditional Rendering Logic
For any list or set that is filtered based on current state:
- [ ] Verify the filter is applied correctly (e.g. already-selected items hidden from picker)
- [ ] Verify the filter updates dynamically as state changes

---

### 12. Toast / Error Message Copy
For every toast or inline error:
- [ ] Verify the exact message text matches the spec
- [ ] Verify the toast is dismissible
- [ ] Verify duplicate toasts do not stack when the same error occurs twice

---

### 13. Sorting / Ordering
For any list that is sorted:
- [ ] Verify items appear in the documented order (e.g. highest count first, alphabetical tiebreak)

---

## What NOT to include in an E2E test plan

Exclude these — they belong in API / unit / integration tests, not Playwright:

- Raw HTTP requests with malformed body fields (missing fields, wrong types)
- Database-level cascade / integrity tests
- Multi-user real-time broadcast/WebSocket tests
- Integer overflow / max-int edge cases
- CSS / tooltip flicker / animation timing

---

## Scenario template

Every scenario in the plan must follow this structure:

```
### N. <concise scenario title>

**Case Type:** Positive | Negative | Edge Case
**Role:** NoAuth | Standard | Problem
**Preconditions:**
- <what must be true before the test runs>

**Steps:**
1. <step>

**Expected Results:**
- <observable outcome>
```
