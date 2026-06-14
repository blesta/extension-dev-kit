# Porting a Blesta Plugin from 5.x to 6.0

> **Audience:** Developers (and AI tools) converting an existing 5.x plugin to run natively in Blesta 6.0.
> Read `create.md` for the full contract; this file is the **diff** — what actually changes in a port.
>
> A plugin port is **mostly UI**. The PHP handler contract (`install`/`upgrade`/`uninstall`/`getActions`/
> `getEvents`/`getCards`/`getPermissions`/`getMessageTemplates`/`cron`/service tabs) is backwards
> compatible. The big job is converting every **admin** page/widget/view to Bootstrap 5 / Paradigm.

---

## 1. Will the old plugin even run?

Largely yes. The `Plugin` base class and `PluginManager` keep the 5.x surface working:

- The handler `get*()` declaration methods are unchanged in name and shape.
- **Actions** accept the **legacy form**: `'action' => 'nav_primary_staff' | 'nav_secondary_staff' |
  'nav_primary_client' | 'nav_secondary_client'` and `'uri' => ...` are auto-mapped by
  `Actions::mapOldFields()` to the new `location`/`url` fields (`nav_staff` / `nav_client`, with
  `nav_primary_client` also seeding `nav_public`). You don't *have* to migrate these, but new code should
  prefer `location`/`url` (see `create.md` §8).
- **Events** still use `['event' => ..., 'callback' => ['this', 'method']]`. Deprecated event **names**
  (e.g. `Invoices.setClosed`, `Services.add`) still fire via `triggerDeprecatedEvent()`, so existing
  listeners keep working; prefer the `*After` variants in new code.
- Controllers/models, the `Plugin.Model` loader namespace, `setDefaultView()`, `$this->uses()`,
  `requireLogin()`, and the `/plugin/...` & `/widget/...` routes are all unchanged.

What needs work is the **admin UI** and a few cosmetics.

---

## 2. The admin/client split governs what converts

Same rule as modules and gateways — **only the admin side converts to Bootstrap 5**:

| Surface | Controllers / views | Host theme (6.0) | Action |
|---|---|---|---|
| **Admin pages** | `admin_*` controllers + their `.pdt` views | **Paradigm** | **Convert to Bootstrap 5** (`reference/common/conversion-ui.md`) |
| **Admin widgets** | `widget_staff_*` action targets (dashboard/client/billing widgets) | **Paradigm** | **Convert to Bootstrap 5** |
| **Manage page** | `admin_manage_plugin.pdt` (Settings > Plugins > Manage) | **Paradigm** | **Convert to Bootstrap 5** |
| **Admin service tabs** | `getAdminServiceTabs()` views | **Paradigm** | **Convert to Bootstrap 5** |
| **Client pages/widgets/cards/tabs** | `client_*` controllers, client dashboard cards, client service tabs | **`bootstrap` portal — still BS4 + jQuery + FontAwesome** | **Leave on BS4** |

The client portal is still Bootstrap 4 + jQuery + FontAwesome in 6.0, so **don't** convert client-facing
views (a BS5 client theme arrives in 6.1). Converting them breaks the client area.

---

## 3. The admin UI conversion (the bulk of the work)

Apply `reference/common/conversion-ui.md` to **every** admin `.pdt` and any admin HTML your handler/cards
emit. The recurring edits:

- **Form controls:** add `'class' => 'form-control'` / `'form-select'` / `'form-check-input'` to every
  `Form` helper call; `form-group` → `mb-3`; legacy `<ul><li>` field rows → `<div class="mb-3">`.
- **Grid/layout:** `col-md-*` offsets and `pull-left/right` → `float-start/end`; `text-right` → `text-end`;
  `btn-default` → `btn-secondary`; `label-*`/`badge` utilities updated.
- **Data attributes:** `data-toggle` → `data-bs-toggle`, `data-target` → `data-bs-target`, `data-dismiss`
  → `data-bs-dismiss`, etc. (the whole `-bs-` infix family).
- **Icons:** `fas fa-…` / `glyphicon-…` → `bi bi-…` in admin views, **and** in action/nav and admin
  service-tab `icon` values. (Cards have **no** icon field — don't add one.)
- **jQuery → vanilla JS.** Paradigm ships **no jQuery**. Rewrite all admin-side `$(...)`,
  `$(document).ready`, `.on('click', …)`, AJAX (`$.ajax`/`$.post`) and DOM toggling to
  `DOMContentLoaded` + `querySelector(All)` + `addEventListener` + `fetch`. This is usually the largest
  single task in a plugin port (admin plugins often carry substantial jQuery).
- **Light *and* dark mode.** Use Paradigm CSS variables (`--border-color`, `--bg-secondary`,
  `--bg-tertiary`) rather than hard-coded colours; verify every admin view in both modes.
- **Plugin-owned assets:** keep shipping your CSS/JS **inside** the plugin (`views/default/css`,
  `views/default/javascript`) and referencing them directly from the `.pdt`. Plugin assets do **not** go
  through core Gulp — don't add them to a build pipeline, and don't touch core assets/SCSS.

> **Widgets need the same treatment as pages.** A `widget_staff_home` / `widget_staff_client` /
> `widget_staff_billing` action renders inside Paradigm, so its view must be BS5 + Bootstrap Icons +
> vanilla JS, light/dark safe — exactly like an admin page.

---

## 4. The PHP pass (small)

- **`config.json` `icon`** — set the **full** Bootstrap Icons class, e.g. `"icon": "bi bi-headset"`
  (`Plugin::getIcon()` returns it verbatim — unlike a *module* icon, nothing prepends `bi`).
- **Actions (optional migration):** you may convert legacy `'action'`/`'uri'` to `'location'`/`'url'` and
  update admin `icon` values to `bi bi-…`. Not required for compatibility — the legacy form is auto-mapped.
- **Events (optional):** migrate hooked names to their `*After` equivalents where a deprecated alias was
  used. Behaviour is unchanged; this just avoids the deprecation path.
- **No handler signature changes.** `install($plugin_id)`, `upgrade($current_version, $plugin_id)`,
  `uninstall($plugin_id, $last_instance)`, `getActions()`, `getEvents()`, `getCards()`,
  `getPermissions()`/`getPermissionGroups()`, `getMessageTemplates()`, `cron($key)`, and the service-tab
  methods keep their 5.x signatures.
- **Cron registration unchanged.** Still `CronTasks->add()` + `addTaskRun()` in `install()`/`upgrade()`,
  executed via `cron($key)` (`create.md` §12).
- **Bump `config.json` `version`** and add an `upgrade()` branch if the port changes schema/settings, so
  existing installs migrate cleanly.

---

## 5. Things that bite during a plugin port

- **AJAX endpoints in admin controllers.** If your 5.x admin pages used jQuery `$.ajax`, the endpoints
  themselves (plugin controller methods) still work — but the **client-side** caller must become `fetch`,
  and POSTs must carry the CSRF token (`$this->Form->getCsrfToken()`). Plugin controller methods are
  directly routable, so you can keep clean per-action endpoints.
- **The Manage page returns HTML.** If your old `AdminManagePlugin` echoed output, make each action
  **return** its rendered partial — `AdminCompanyPlugins::manage()` captures the return value as the page
  content and primes the controller via `$this->parent` (use `$this->parent->requireLogin()` /
  `$this->parent->structure->set('page_title', …)`). See `create.md` §9.
- **Declaration arrays overwrite on upgrade.** `getActions()`/`getCards()`/`getEvents()`/
  `getMessageTemplates()`/`getPermissions()`/`getPermissionGroups()` are re-read and **replace** existing
  registrations on upgrade. Ensure each returns the **complete** current set, not a delta, or you'll drop
  registrations on the version bump.
- **Permission groups before permissions.** Keep groups and permissions consistent; PluginManager re-adds
  both on upgrade (groups first).
- **Icon contexts differ.** `config.json` icon = full `bi bi-…`; admin action/nav and admin service-tab
  icons = full `bi bi-…`; **client** service-tab icons stay `fas fa-…` for 6.0. Cards carry no icon field.

---

## 6. What does *not* change

- The `Plugin` base contract and every `get*()` method name/shape.
- The `/admin/plugin/…`, `/client/plugin/…`, `/plugin/…`, `/widget/…` routing.
- `Plugin.Model` loader namespacing; controllers extending `AppController`; `setDefaultView()`.
- The event system (`EventInterface`, `['this','method']` callbacks, before/after pattern).
- Cron task/run registration and `cron($key)`.
- ACL alias/action semantics; messenger template shapes.
- `language/en_us/` layout and the `!error.` convention.

---

## 7. Conversion checklist

- [ ] **Admin** pages, widgets, the Manage page, and admin service tabs converted to Bootstrap 5 / Paradigm
      (`reference/common/conversion-ui.md`), Bootstrap Icons, **vanilla JS**, light **and** dark mode.
- [ ] **Client** pages/widgets/cards/tabs left on BS4 + jQuery + FontAwesome.
- [ ] All admin-side jQuery (including `$.ajax`) rewritten to vanilla `fetch`/`addEventListener`; CSRF token
      added to AJAX POSTs.
- [ ] `config.json` `"icon"` set to the full `bi bi-…` class; admin action/nav and admin service-tab icons
      → `bi bi-…`; client service-tab icons unchanged (cards have no icon field).
- [ ] `AdminManagePlugin` actions **return** their HTML and use `$this->parent` for login/title.
- [ ] Each `get*()` declaration returns the **complete** set (verified against the upgrade-overwrites
      behaviour).
- [ ] *(Optional)* legacy `action`/`uri` migrated to `location`/`url`; deprecated event names moved to
      `*After`.
- [ ] `config.json` `version` bumped and an `upgrade()` branch added for any schema/settings change.
- [ ] No handler/controller/model signature changes; cron registration untouched.
- [ ] Plugin-owned CSS/JS still shipped inside the plugin and referenced directly (no core Gulp, no core
      asset edits).

---

## 8. See also

- `create.md` — the full plugin contract: handler, controllers/routing, models, actions, the Manage page,
  cards, events, cron, permissions, message templates, and service tabs.
- `reference/common/conversion-ui.md` — the admin Bootstrap 4→5 / jQuery→vanilla / FA→Bootstrap Icons guide
  (shared by all extension types).
