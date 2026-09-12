---
name: playwright-e2e-testing
description: Use when writing Playwright E2E tests. Probe real DOM first.
---

# Playwright E2E Test Authoring

Writing browser E2E tests that pass against a live app. The core discipline: the
rendered DOM is the source of truth, not the template/Blade source. JS frameworks
and component libs (Livewire, MaryUI, Alpine) transform markup into nodes you can
only know by looking at the running page.

## Workflow (in order)

1. **Baseline first.** Run the existing spec before touching it, e.g.
   `npx playwright test tests/e2e/<dir>/ --reporter=list`. You want to see what
   *actually* passes vs. what the plan claims, and catch environment issues (missing
   browser, wrong baseURL) before writing anything new.

2. **Probe the real DOM with a throwaway script.** Before writing or fixing a
   locator, run a short Node script (`import { chromium } from '@playwright/test'`)
   that drives the page and prints `innerText()`, `.count()`, and
   `evaluateAll()` of element attributes. Delete it after. This turns "guess the
   selector" into "read the selector off the rendered node". See
   `references/livewire-maryui-quirks.md` for the attribute shapes to expect.

3. **Write locators from the probe output.** Prefer, in order: stable `#id`,
   `[wire\:model="..."]` attribute selectors, `getByRole('name', { name: '…' })`,
   visible Persian/RTL text. Do NOT assume `data-testid` exists — check first.

4. **Run green, then commit + push** per the repo's convention.

## Pitfalls (each cost real time)

- **Validation/error text renders in TWO places** (inline under the field AND in a
  summary/`x-errors` box). A bare `locator('text=...')` then throws strict-mode
  violation. Use `.first()` or `toContainText` on a scoped locator.
- **URL assertions against a `baseURL`:** `toHaveURL('**/login')` is unreliable —
  the glob mixes with the base into a wrong absolute URL. Use a predicate:
  `toHaveURL((url) => url.pathname === '/login')` (and `waitForURL` the same way).
- **`required` fields trigger native HTML5 validation, not server-side.** An empty
  submit stays on the page with `validity.valueMissing === true` and NO round-trip,
  so there is no Persian "required" message to assert. Assert the validity flag
  (`toHaveJSProperty('validity.valueMissing', true)`) instead of expecting a
  translated error string.
- **Stateful side effects must be reverted in the same test.** A password-change
  test that alters a shared seeded account's password must change it back inside
  the test, or the next run starts with a broken login.
- **`npm install` bumps patch versions in package-lock.json** (unrelated deps get
  `^`-range updates). That diff is real and harmless — commit it, don't fight it.
- **Never hardcode test credentials in source.** Use a `.env.test` file (gitignored)
  and `process.env.TEST_N_CODE` / `process.env.TEST_PASSWORD` with fallback defaults
  in the shared fixtures module. Credentials in a public repo are a security leak.
  Add `dotenv` to devDependencies and load `.env.test` in `playwright.config.ts`.
- **Replace `waitForTimeout` with reactive waits.** Hardcoded sleeps are the #1
  source of flaky tests. For Livewire: use `page.waitForFunction(() =>
  !document.querySelector('.wire-loading'))` to wait for request completion. For
  search results: use `page.waitForSelector(selector, { state: 'visible' })`. For
  dialog handling where no DOM signal exists, keep `waitForTimeout` but add a
  comment explaining why. See `references/livewire-maryui-quirks.md` for the
  `.wire-loading` pattern.
- **Do NOT use `networkidle` with Livewire SPAs.** Livewire maintains persistent
  connections; `networkidle` may fire too early or never resolve. Wait for specific
  UI state instead: `waitForSelector('table tbody tr')` or
  `waitForFunction(() => !document.querySelector('.wire-loading'))`.
- **Register dialog handlers BEFORE triggering the action.** If you set up
  `page.on('dialog', handler)` after clicking the button that triggers it, the
  dialog may fire before the listener is registered on slow machines.
- **Smoke-link loop: use `page.request.get`, not page navigation.** To verify "no
  broken links", collect hrefs once from the drawer (`evaluateAll` deduping + filtering
  `href.startsWith('/') && !href.startsWith('//')`), then issue each as
  `page.request.get(href)` — the APIRequestContext shares the browser's auth
  session/cookies. Only `status >= 500` is a real break; 302 → /login and 404 mean
  "route exists but redirects/missing", not a crash, so tolerate them.
