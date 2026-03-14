---
name: e2e-reviewer
description: Use when reviewing, auditing, or improving E2E test specs for Playwright, Cypress, or Puppeteer — static code analysis of existing test files, not diagnosing runtime failures. Triggers on "review my tests", "audit test quality", "find weak tests", "my tests always pass but miss bugs", "tests pass CI but miss regressions", "improve playwright tests", "improve cypress tests", "check test coverage gaps", "my tests are fragile", "tests break on every UI change", "test suite is hard to maintain", "we have coverage but bugs still slip through". Detects 10 anti-patterns -- name-assertion mismatch, missing Then, error swallowing, always-passing assertions, bypass patterns (conditional assertions + force:true), raw DOM queries, duplicate scenarios, hard-coded sleeps, flaky test patterns (positional selectors + serial ordering), and YAGNI in Page Objects.
---

# E2E Test Scenario Quality Review

Systematic checklist for reviewing E2E **spec files AND Page Object Model (POM) files**. Framework-agnostic principles; code examples show Playwright, Cypress, and Puppeteer where they differ.

## Phase 1: Automated Grep Checks (Run First)

Once the review target files are determined, use the Grep tool to mechanically detect known anti-patterns **before** LLM analysis. Run each check below against the `e2e/` directory (or equivalent). Replace `e2e/` with the actual test directory if different.

**What each check detects:**

- **#3 Error Swallowing** — `.catch(() => {})` or `.catch(() => false)` silently hides failures. Search `.ts/.js/.cy.*` for `\.catch\(\s*(async\s*)?\(\)\s*=>`, excluding `node_modules` and lines with `// JUSTIFIED`. **Critical:** `// JUSTIFIED:` must appear on the **same line** as `.catch(` — a comment on the next line is invisible to grep and does NOT suppress the flag.
- **#4 Always-Passing** — assertions that can never fail. Search for `toBeGreaterThanOrEqual(0)` or `should.*(gte|greaterThan).*0`. Also search `.ts/.js/.cy.*` (including POM/util files, not just specs) for `toBeAttached()` — flag every hit with no `// JUSTIFIED:` on the same line for manual review.
- **#5 Bypass Patterns** — two sub-patterns that suppress what the framework would normally catch: (a) `expect()` inside `if(isVisible)` silently skips assertions — search `.spec.*/.test.*/.cy.*` for `if.*(isVisible|is\(.*:visible.*\))`; (b) `{ force: true }` bypasses actionability checks (visibility, enabled state) — search `.ts/.js/.cy.*` for `force:\s*true`. Exclude lines with `// JUSTIFIED`. **Note:** The `if(isVisible)` grep covers spec files only — review POM helper methods manually in Phase 2.
- **#6 Raw DOM Queries** — `document.querySelector` bypasses framework auto-wait. Search `.spec.*/.test.*/.cy.*` for `document\.querySelector` (covers both `evaluate()` and `waitForFunction()`).
- **#8 Hard-coded Sleeps** — explicit sleeps cause flakiness. Search `.ts/.js/.cy.*` for `waitForTimeout` or `cy\.wait\(\d`.
- **#9 Flaky Test Patterns (partial)** — two sub-patterns that cause CI instability: (a) positional selectors `nth()`, `first()`, `last()` without explanation — search `.spec.*/.test.*/.cy.*` for `\.nth\(|\.first\(\)|\.last\(\)`; (b) `test.describe.serial()` creates order-dependent tests that break parallel sharding `[Playwright only]` — search `.spec.*/.test.*` for `\.describe\.serial\(`. Exclude lines with `// JUSTIFIED`.

**Interpreting results:**
- Zero hits → no mechanical issues found, proceed to Phase 2
- Any hit → report each line as an issue (includes file:line)
- Lines with `// JUSTIFIED` comments are intentional — skip them

**Output Phase 1 results as-is.** The LLM must not reinterpret them.

---

## Phase 2: LLM Review (Subjective Checks Only)

Patterns already detected in Phase 1 (#3, #4, #5, #6, #8, #9 partial) are **skipped**.
The LLM performs only these checks:

| # | Check | Reason |
|---|-------|--------|
| 1 | Name-Assertion Alignment | Requires semantic interpretation |
| 2 | Missing Then | Requires logic flow analysis |
| 7 | Duplicate Scenarios | Requires similarity comparison |
| 9 | Flaky Test Patterns | Requires context judgment for nth() and serial ordering |
| 10 | YAGNI in POM | Requires usage grep then judgment |

---

## Phase 3: Coverage Gap Analysis (After Review)

After completing Phase 1 + 2, identify scenarios the test suite does NOT cover. Scan the page/feature under test and flag missing:

| Gap Type | What to look for |
|----------|-----------------|
| Error paths | Form validation errors, API failure states, network offline, 404/500 pages |
| Edge cases | Empty state, max-length input, special characters, concurrent actions |
| Accessibility | Keyboard navigation, screen reader labels, focus management after modal/dialog |
| Auth boundaries | Unauthorized access redirects, expired session handling, role-based visibility |

**Output:** List up to 5 highest-value missing scenarios as suggestions, not requirements. Format:

```markdown
## Coverage Gaps (Suggestions)
1. **[Error path]** No test for form submission with server error — add API mock returning 500
2. **[Edge case]** No test for empty list state — verify empty state message shown
```

---

## Review Checklist

Run each check against every **non-skipped** test and every **changed POM file**.

**Important:** `test.skip()` with a reason comment or reason string is intentional — do NOT flag or remove these. Only flag mid-test conditional skips that hide failures (see #5).

---

### Tier 1 — P0/P1 (always check)

#### 1. Name-Assertion Alignment `[LLM-only]`

**Symptom:** Test name promises something the assertions don't verify.

```typescript
// BAD — name says "status" but only checks visibility
test('should display user status', () => {
  await expect(status).toBeVisible();  // no status content check
});
```

**Rule:** Every noun in the test name must have a corresponding assertion. Add it or rename.

**Procedure:**
1. Extract all nouns from the test name (e.g., "should display user **status**")
2. For each noun, search the test body for `expect()` that verifies it
3. Missing noun → add assertion or remove noun from name

**Common patterns:** "should display X" with only `toBeVisible()` (no content check), "should update X and Y" with assertion for X but not Y, "should validate form" with only happy-path assertion.

#### 2. Missing Then `[LLM-only]`

**Symptom:** Test acts but doesn't verify the final expected state.

```typescript
// BAD — toggles but doesn't verify the dismissed state
test('should cancel edit on Escape', () => {
  await input.click();
  await page.keyboard.press('Escape');
  await expect(text).toBeVisible();
  // input still hidden?
});
```

**Rule:** For toggle/cancel/close actions, verify both the restored state AND the dismissed state.

**Procedure:**
1. Identify the action verb (toggle, cancel, close, delete, submit, undo)
2. List the expected state changes (element appears/disappears, text changes, count changes)
3. Check that BOTH sides of the state change are asserted

**Common patterns:** Cancel/Escape without verifying input is hidden, delete without verifying count decreased, submit without verifying form resets, tab switch without verifying previous tab content is hidden.

#### 3. Error Swallowing `[grep-detectable]`

**Symptom (spec):** `try/catch` wrapping assertions — test passes on error.

**Symptom (POM):** `.catch(() => {})` or `.catch(() => false)` on awaited operations — caller never sees the failure.

```typescript
// BAD spec — silent pass
try { await expect(header).toBeVisible(); }
catch { console.log('skipped'); }

// BAD POM — caller thinks execution succeeded
await loadingSpinner.waitFor({ state: 'detached' }).catch(() => {});
```

**Rule (spec):** Never wrap assertions in try/catch. Use `test.skip()` in `beforeEach` if the test can't run.

**Rule (POM):** Remove `.catch(() => {})` / `.catch(() => false)` from wait/assertion methods. If the operation can legitimately fail, the caller should decide how to handle it. Only keep catch for UI stabilization like `input.click({ force: true }).catch(() => textarea.focus())`.

#### 4. Always-Passing Assertions `[grep-detectable + LLM confirmation]`

**Symptom:** Assertion that can never fail.

```typescript
// BAD — count >= 0 is always true
expect(count).toBeGreaterThanOrEqual(0);

// SUSPECT — element may always be in DOM; needs review
await expect(page.locator('.app-shell')).toBeAttached();
```

**Rule:** Search for `toBeGreaterThanOrEqual(0)` and `toBeAttached()`. For `toBeAttached()` hits with no `// JUSTIFIED:` comment on the same line, confirm whether the element can ever be absent from the DOM. If it is unconditionally rendered or always present in the static HTML shell, the assertion is vacuous → flag P0. If `// JUSTIFIED:` explains the element is intentionally CSS-hidden (`visibility:hidden`, not `display:none`), skip.

**Fix:** Replace `toBeGreaterThanOrEqual(0)` with `toBeGreaterThan(0)`. Replace vacuous `toBeAttached()` with `toBeVisible()`, or remove if other assertions already cover the element.

#### 5. Bypass Patterns `[grep-detectable]`

Two sub-patterns that suppress what the framework would normally catch — making tests pass when they should fail.

**5a. Conditional assertion bypass** — `expect()` inside `if` block or mid-test `test.skip()`.

```typescript
// BAD — if spinner never appears, assertion never runs
if (await spinner.isVisible()) {
  await expect(spinner).toBeHidden({ timeout: 5000 });
}
```

**Rule:** Every test path must contain at least one `expect()`. Move environment checks to `beforeEach` or declaration-level `test.skip()`.

**5b. Force true bypass** — `{ force: true }` skips actionability checks (visibility, enabled state, pointer-events), hiding real UX problems that real users would encounter.

```typescript
// BAD — hides overlay or disabled state
await page.click('[data-test="submit"]', { force: true });

// GOOD — wait for the real condition
await page.locator('.overlay').waitFor({ state: 'detached' });
await page.click('[data-test="submit"]');
```

**Rule:** Each `{ force: true }` must have `// JUSTIFIED:` on the same line explaining why the element is not normally actionable (e.g. contenteditable, shadow DOM). Without a comment, flag P1.

#### 6. Raw DOM Queries (Bypassing Framework API) `[grep-detectable]`

**Symptom:** Test uses `document.querySelector*` / `document.getElementById` inside `evaluate()` or `waitForFunction()` when the framework's element API could do the same job.

```typescript
// BAD — waitForFunction with querySelector; use locator.waitFor() instead
await page.waitForFunction(
  () => document.querySelectorAll('.item').length > 0
);

// BAD — evaluate with querySelector; returns stale boolean, no auto-wait
const has = await page.evaluate(() => !!document.querySelector('.result'));
expect(has).toBe(true);
```

```typescript
// GOOD — framework API with auto-wait and retry
await page.locator('.item').waitFor({ state: 'attached' });
await expect(page.locator('.result')).toBeVisible();
```

**Why it matters:** No auto-waiting, no retry, boolean trap, framework error messages lost.

**Rule:** Use the framework's element API instead of raw DOM:
- **Playwright:** `locator.waitFor({ state: 'attached' })` replaces `waitForFunction(() => querySelector(...) !== null)`; `page.locator()` + web-first assertions replaces `evaluate(() => querySelector(...))`
- **Cypress:** `cy.get()` / `cy.find()` — avoid `cy.window().then(win => win.document.querySelector(...))`
- **Puppeteer:** `page.$()` / `page.waitForSelector()` — avoid `page.evaluate(() => document.querySelector(...))`

Only use `evaluate`/`waitForFunction` when the framework API genuinely can't express the condition: multi-condition AND/OR logic, `getComputedStyle`, `children.length`, cross-element DOM relationships, or `body.textContent` checks. Add `// JUSTIFIED:` explaining why.

---

### Tier 2 — P1/P2 (check when time permits)

#### 7. Duplicate Scenarios (DRY) `[LLM-only]`

**Symptom:** Two tests share >70% of their steps with minor variations. Or an entire spec file's tests are all subsets of another spec file (zombie spec file).

**Rule (within file):** Merge tests that differ only in setup or a single assertion. Use the richer verification set from both.

**Rule (cross-file):** After reviewing all files in scope, cross-check tests with similar names across different spec files. If test A in one file is a subset of test B in another, delete A and strengthen B. If all tests in file A are subsets of tests in file B, delete file A entirely (zombie spec file).

**Procedure:**
1. List all test names in the file — look for similar prefixes or overlapping verbs
2. For each pair with >70% step overlap, compare their assertion sets
3. If one is a subset of the other, delete the weaker test and keep the richer one
4. Cross-check spec file names — if two files cover the same feature, compare test-by-test for complete duplication

**Common patterns:** "should add item" and "should add item and verify count" (subset), "should open dialog" in file A and "should open dialog and fill form" in file B (cross-file subset), parameterizable tests written as separate cases, a `feature-basic.spec.ts` entirely covered by `feature-full.spec.ts`.

#### 8. Hard-coded Sleeps `[grep-detectable]`

**Symptom:** Explicit sleep calls pause execution for a fixed duration instead of waiting for a condition.

```typescript
// BAD — arbitrary sleep
await page.waitForTimeout(2000);

// BAD — Cypress explicit wait
cy.wait(3000);
```

**Rule:** Never use explicit sleep (`waitForTimeout` / `cy.wait(ms)`) — rely on framework auto-wait or condition-based waits. Replace with `locator.waitFor()`, `expect(locator).toBeVisible()`, or `cy.get(selector).should('be.visible')`.

Note: `timeout` option values in `waitFor({ timeout: N })` or `toBeVisible({ timeout: N })` are NOT flagged — these are bounds, not sleeps.

#### 9. Flaky Test Patterns `[LLM-only + grep]`

Two sub-patterns that cause tests to fail intermittently in CI or parallel runs.

**9a. Positional selectors** — `nth()`, `first()`, `last()` without a comment break when DOM order changes.

```typescript
// BAD — breaks if DOM order changes
await expect(items.nth(2)).toContainText('Settings');
```

**Rule:** Prefer `data-testid`, role-based, or attribute selectors. If `nth()` is unavoidable, add `// JUSTIFIED:` explaining why.

**9b. Serial test ordering** `[Playwright only]` — `test.describe.serial()` makes tests order-dependent: a single failure cascades to all subsequent tests, and the suite can't be sharded.

```typescript
// BAD — second test implicitly relies on cart state from first
test.describe.serial('checkout flow', () => {
  test('add item to cart', async ({ page }) => { ... });
  test('complete checkout', async ({ page }) => { ... });
});

// GOOD — self-contained
test('add item and complete checkout', async ({ page }) => {
  await addItemToCart(page);
  await completeCheckout(page);
  await expect(page.locator('.confirmation')).toBeVisible();
});
```

**Rule:** Replace serial suites with self-contained tests or independent tests using `beforeEach` for shared setup. If sequential flow is genuinely required, use a single test with `test.step()` blocks.

#### 10. YAGNI in Page Objects `[LLM-only]`

**Symptom:** POM has locators or methods never referenced by any spec. Or a POM class extends a parent with zero additional members (empty wrapper class).

**Procedure:**
1. List all public members of each changed POM file
2. Grep each member across all test files and other POMs
3. Classify: USED / INTERNAL-ONLY (`private`) / UNUSED (delete)
4. Check if any POM class has zero members beyond what it inherits — empty wrappers add no value unless the convention is intentional

**Common patterns:** Convenience wrappers (`clickEdit()` when specs use `editButton.click()`), getter methods (`getCount()` when specs use `toHaveCount()`), state checkers (`isVisible()` when specs assert on locators directly), pre-built "just in case" locators, empty subclass created for future expansion.

**Rule:** Delete unused members. Make internal-only members `private`. When creating new shared utils, ensure they will be used by 2+ specs. Flag empty wrapper classes for review — they may be intentional convention or dead code.

**Output:**
```
| File | Member | Used In | Status |
|------|--------|---------|--------|
| modal-page.ts | openModal() | (none) | DELETE |
| modal-page.ts | closeButton | internal only | PRIVATE |
| search-page.ts | (class body empty) | — | REVIEW |
```

---

## Output Format

Present findings grouped by severity:

```markdown
## [P0/P1/P2] Task N: [filename] — [issue type]

### N-1. `[test name or POM method]`
- **Issue:** [description]
- **Fix:** [name change / assertion addition / merge / deletion]
- **Code:**
  ```typescript
  // concrete code to add or change
  ```
```

**After all findings, append a summary table:**

```markdown
## Review Summary

| Sev | Count | Top Issue | Affected Files |
|-----|-------|-----------|----------------|
| P0  | 3     | Missing Then | auth.spec.ts, form.spec.ts |
| P1  | 5     | Duplicate Scenarios | settings.spec.ts |
| P2  | 2     | Hard-coded Sleeps | dashboard.spec.ts |

**Total: 10 issues across 4 files. Fix P0 first.**
```

**Severity classification:**
- **P0 (Must fix):** Test silently passes when the feature is broken — no real verification happening
- **P1 (Should fix):** Test works but gives poor diagnostics, wastes CI time, or misleads developers
- **P2 (Nice to fix):** Weak but not wrong — maintenance and robustness improvements

## Quick Reference

| # | Check | Sev | Phase | Detection Signal |
|---|-------|-----|-------|-----------------|
| 1 | Name-Assertion | P0 | LLM | Noun in name with no matching `expect()` |
| 2 | Missing Then | P0 | LLM | Action without final state verification |
| 3 | Error Swallowing | P0 | grep | `try/catch` in spec, `.catch(() => {})` in POM |
| 4 | Always-Passing | P0 | grep+LLM | `>=0`; `toBeAttached()` with no `// JUSTIFIED:` → confirm if element can be absent |
| 5 | Bypass Patterns | P0/P1 | grep | `expect()` inside `if`; `force: true` without `// JUSTIFIED:` |
| 6 | Raw DOM Queries | P1 | grep | `document.querySelector` in `evaluate` |
| 7 | Duplicate Scenarios | P1 | LLM | >70% shared steps; entire spec file is a subset of another |
| 8 | Hard-coded Sleeps | P2 | grep | `waitForTimeout()`, `cy.wait(ms)` |
| 9 | Flaky Test Patterns | P1 | LLM+grep | `nth()` without comment; `test.describe.serial()` |
| 10 | YAGNI in POM | P2 | LLM | Public member not referenced in any spec; empty wrapper class |

---

## Suppression

When a grep-detected pattern is intentional, add a `// JUSTIFIED: [reason]` comment to the line. Phase 1 will exclude it.

**Critical: the comment must be on the exact same line as the pattern.** A comment on the preceding or following line is NOT detected by grep.

```typescript
// BAD — comment is inside the catch block; grep still flags the .catch( line
await input.click({ force: true }).catch(async () => {
  // JUSTIFIED: UI stabilization
  await textarea.focus();
});

// GOOD — comment is on the same line as .catch(
await input.click({ force: true }).catch(async () => { // JUSTIFIED: UI stabilization
  await textarea.focus();
});

// GOOD — single-line catch
await input.click({ force: true }).catch(() => {}); // JUSTIFIED: UI stabilization — click attempt follows
```

**Named function wrappers do not help.** Extracting a `.catch()` into a named function still requires `// JUSTIFIED:` on each individual `.catch(` line inside that function:

```typescript
// STILL NEEDS justified on each .catch( line inside
const tryFocus = async () => {
  await el.waitFor({ state: 'visible' }).catch(() => {}); // JUSTIFIED: UI stabilization
  await el.click().catch(async () => { // JUSTIFIED: UI stabilization — falls back to focus
    await el.locator('textarea').focus();
  });
};
```

Example: `await input.click({ force: true }).catch(() => textarea.focus()); // JUSTIFIED: UI stabilization fallback`
