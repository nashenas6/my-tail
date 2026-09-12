# Livewire + MaryUI selector quirks

Concrete DOM shapes to expect when testing a Livewire 4 / MaryUI (DaisyUI) / Alpine app
so you can write locators from a probe instead of guessing from Blade templates.

## Inputs: use `wire:model`, not `id`

MaryUI `<x-input>` renders an actual `<input>` whose `id` is a MaryUI-prefixed random
hash (e.g. `mary5eded4c94f55bcb30b46b631f70d006ecurrentPassword`). The stable, predictable
attribute is `wire:model`, and it survives the render:

```ts
// in the probe: dump what's there
await page.locator('input[type="password"]').evaluateAll(els =>
  els.map(e => ({ id: e.id, wireModel: e.getAttribute('wire:model') })));

// then select by it
const el = (wm) => page.locator(`input[wire\:model="${wm}"]`);  // note the escaped colon
```

Plain hand-written forms (e.g. a login page) keep stable `#id`/`#password` — use those.

## Buttons: watch for duplicate `type=submit`

A MaryUI layout puts a logout `<form method=POST action=/logout>` with its own submit
button (`icon="o-power"`, tooltip `logoff`) in the sidebar alongside the page's real
submit. A bare `button[type="submit"]` resolves to TWO elements → strict-mode violation.
Disambiguate with `getByRole('button', { name: 'تغییر رمز' })` (visible label) or scope
to `form[action*="logout"] button[type="submit"]` for logout.

## Logout is a POST form, not a dropdown

There is no `[data-theme-toggle]`/`.dropdown` logout menu. The real control is
`form[action*="logout"] button[type="submit"]`. Clicking it POSTs and redirects to
`/login`.

## Validation messages are NOT always translated

MaryUI field labels / server-side `ValidationException` messages (e.g. `رمز فعلی اشتباه است.`)
come through, but some Laravel validation strings (like the `min` rule) surface with the
attribute name untranslated, e.g. `new password باید حداقل 8 کاراکتر باشد.` and
`new password confirmation و new password باید مطابقت داشته باشند.`. Assert on the
stable Persian fragment (`حداقل 8 کاراکتر`, `مطابقت داشته باشند`) rather than a full
translated sentence.

## Sidebar / menu (`x-menu activate-by-route`)

The sidebar is `<x-menu activate-by-route>` inside a collapsible drawer
(`#main-drawer` checkbox, `<x-slot:sidebar drawer="main-drawer">`). DOM facts:

- **Collapsible sections render as `<details>/<summary>`, not dropdowns.** Each
  `<x-menu-sub>` becomes a `<details>` whose `<summary>` holds the Persian section
  title. Submenu items are `<li><a wire:navigate>` inside. Expand/collapse = toggling
  `details` `open` — click the `<summary>`, then assert the child `ul a` visibility.
- **Active item = `a.mary-active-menu` + `data-current` attr**, NOT `.menu-active`.
  `activate-by-route` adds `bg-base-300` styling too. Assert
  `locator('a.mary-active-menu')` and compare its `href` against the current route.
- **There is no header dropdown.** The profile/account link is a direct
  `a[href="/profile"]`. Logout lives only in the sidebar (`form[action*=logout]`).
- **The top `x-nav` header is `lg:hidden`** (mobile-only). Its profile/search links
  are `hidden` in a desktop viewport — header-assertion tests must set a mobile
  viewport (`test.use({ viewport: { width: 390, height: 844 } })`).

### Drawer label ambiguity (checkbox-drawer pattern)

`label[for="main-drawer"]` resolves to TWO elements: the hamburger opener
(`<label for="main-drawer" class="lg:hidden">`) AND the overlay closer
(`<label for="main-drawer" class="drawer-overlay" aria-label="close sidebar">`).
Both target the same `#main-drawer` checkbox → strict-mode violation. Disambiguate:
`label[for="main-drawer"]:not(.drawer-overlay)` (the hamburger) vs
`.drawer-overlay` (the scrim).

### Mobile drawer auto-collapses after navigation

Clicking a menu item on mobile navigates via `wire:navigate` and the drawer
checkbox resets (closes). The active sidebar item is then `hidden` in the DOM.
After a mobile navigation, assert the URL/page — NOT `.toBeVisible()` on the
active menu element.

## Toast success message

MaryUI `Toast` trait renders a `.toast` element (class `toast ... toast-top toast-end`).
Assert the success text with `toContainText(...)` + `.first()` on `.toast` to avoid
strict-mode if concurrent toasts accumulate.

## Reactive waits for Livewire requests

Livewire adds a `.wire-loading` class to the page during pending requests. Use this
instead of `waitForTimeout`:

```ts
// Wait for a Livewire request to complete
await page.waitForFunction(() => {
  return !document.querySelector('.wire-loading') ||
         document.querySelectorAll('.wire-loading[style*="display: none"]').length > 0;
}, { timeout: 15000 });

// Wait for debounced search results to appear (e.g. person search dropdown)
await page.waitForSelector('div.max-h-40 div.p-2', { state: 'visible', timeout: 10000 });

// Wait for a table to re-render after filter change
await page.waitForSelector('table tbody tr', { timeout: 10000 });
```

For search inputs with Livewire debounce (`wire:model.live.debounce.500ms`), wait
for the result container to become visible rather than sleeping for an arbitrary
duration. The debounce delay varies with server load.
