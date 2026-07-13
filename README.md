# Delegarity

Delegarity is a Tampermonkey/Greasemonkey userscript that augments the CA Clarity Timesheet Browser screen by injecting a "My Delegees" widget and wiring it into Clarity's existing filter workflow.

## Runtime Flow

When the userscript loads, it does not immediately modify Clarity components. It first stages static assets, then waits for the Clarity SPA shell lifecycle to reach the right moment.

### 1. Script activation

- The userscript is active only on the base Clarity URL matched in metadata.
- It runs at `document-idle`, after the page and required libraries are available.
- External runtime dependencies are loaded through metadata requirements:
  - `inject-some.js` for resource injection helpers
  - Handlebars for client-side templating

### 2. Bundled resource staging

- The script reads bundled `@resource` files with `GM_getResourceText`.
- Those resources are injected into the live page:
  - CSS via `injectsome.content.css(...)`
  - HTML templates via `injectsome.content.html(...)`
  - Core logic and feature logic via `injectsome.content.script(...)`

This keeps deployment self-contained: stylesheet, templates, and logic ship in one userscript package.

### 3. Hooking into the Clarity SPA lifecycle

- Delegarity listens to RequireJS module loading through `requirejs.onResourceLoad`.
- Once Clarity shell module `uitk/js/shell` loads, it wraps `clarity.uitk.shell.render` with a post-render callback.
- After every shell render, it inspects the routed page descriptor and only proceeds when page id is `timeadmin.timesheetBrowser`.

This is the activation gate for the feature: no timesheet browser route, no Delegarity UI injection.

### 4. Route and workspace readiness guard

- `delegarity.timeadmin.timesheetbrowser(...)` checks the route hash for the expected action.
- `delegarity.onClarityReady(...)` waits for the Clarity workspace container (`ppm_workspace`) and confirms URL alignment before executing injection.

This avoids racing against partially rendered SPA content.

### 5. Widget injection and enhancement

On the Timesheet Browser page, Delegarity:

- Locates existing Clarity filter section/table nodes.
- Inserts a dedicated cell (`delegarityContainer`) into the native filter row.
- Renders Handlebars templates (`delegee-widget`, `delegee-partial`) into that container.
- Wires row actions for:
  - fill resource filter fields
  - fill and submit filter (`submitForm(...)`)
  - remove delegee from storage
  - open resource details

It also enhances the Resource Name field with:

- A "Select My Delegee" action icon
- jQuery UI autocomplete backed by Clarity lookup endpoint

### 6. Persistence model

- Delegees are stored in `localStorage` under a page-specific key.
- Data is normalized into `{ key, value, unique }` items.
- UI is re-rendered from storage after add/remove to stay stateless between interactions.

## Why Injected UI Feels Native

Delegarity appears native because it does not create an isolated app shell. It mounts directly inside Clarity's own DOM and conventions:

- Uses Clarity table/filter layout and inserts into existing filter row structure.
- Reuses Clarity classes and sprite assets (`ui/uitk/images/s.gif`, `fields.png`) for icons and button visuals.
- Calls existing global Clarity functions (`submitForm`) instead of replacing workflow.
- Reads/writes the same filter inputs Clarity already uses (`ff_res_name`, `ff_unique_name`).
- Runs inside the same JavaScript context, sharing jQuery, RequireJS lifecycle, and page globals.

Because behavior and styling are attached to native structures, Clarity treats the injected parts as ordinary page elements.

## Key Files

- `delegarity.user.js`: userscript metadata, resource injection, and shell render hook.
- `delegarity.core.js`: shared helpers (Clarity readiness + XHR diagnostics).
- `timeadmin.timesheet.js`: timesheet browser feature logic, widget rendering, events, storage.
- `timeadmin.timesheet.html`: Handlebars templates for widget and rows.
- `delegarity.css`: icon styling that aligns with Clarity sprite assets.
