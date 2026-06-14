# Porting a WHMCS Addon Module to a Blesta 6 Plugin

> **Audience:** Developers (and AI tools) converting a **WHMCS addon module** (`modules/addons/{name}/`)
> into a native **Blesta 6.0 plugin**.
>
> **Read `create.md` first** — it is the Blesta-side plugin contract (handler, directly-routable
> controllers/models, `getActions`, events, cron, ACL, cards, the "Manage" page, service tabs). This file is
> the **WHMCS→Blesta mapping**.
>
> **Sources of truth:** the WHMCS SDK `sample-addon-module` (`addonmodule.php`, `lib/Admin/*`,
> `lib/Client/*`, `templates/*.tpl`, `hooks.php`) and `developers.whmcs.com/addon-modules/`. The Blesta side
> is `components/plugins/lib/plugin.php`, `app/models/plugin_manager.php`, and shipped plugins
> (`support_manager`, `order`, `feed_reader`).
>
> **Shared WHMCS conventions** (licensing caveat, the re-source-don't-transform rule, config field-type
> table, error/logging translation, Smarty→`.pdt`, naming/packaging) live in
> **`reference/common/whmcs-mapping.md`** — read it once; this doc covers only what's addon-specific.

---

## 0. The mental model — a WHMCS addon *is* a Blesta plugin

Of the WHMCS extension types, the **addon** is the one that maps to a **Blesta plugin** (the provisioning
and registrar modules map to Blesta *modules* — see `reference/modules/convert-from-whmcs.md`). A WHMCS
addon and a Blesta plugin do the same job: **extend the application** with admin pages, client pages, custom
tables, settings, hooks/events, and ACL.

| | **WHMCS addon** | **Blesta plugin** |
|---|---|---|
| Shape | procedural `addonmodule_*($vars)` + `lib/Admin`, `lib/Client` dispatchers/controllers | `class NamePlugin extends Plugin` (handler) **+** directly-routable controllers/models |
| Lifecycle | `_activate` / `_deactivate` / `_upgrade` | `install($id)` / `uninstall($id, $last)` / `upgrade($ver, $id)` |
| Admin page | `_output($vars)` echoes HTML (often via a hand-rolled dispatcher) | an **admin controller** (`admin_main`) reached by a `nav_staff` action — the router dispatches methods for you |
| Client page | `_clientarea($vars)` returns a page array | a **client controller** (`client_main`) reached by a `nav_client` action |
| Settings | `_config()` `fields[]` auto-rendered by WHMCS | you build a settings form (the **"Manage" page** or a settings controller) |
| Hooks | `hooks.php` + `add_hook()` | `getEvents()` on the handler |
| Access control | admin-role access + `requirelogin` | `getPermissions()`/`getPermissionGroups()` (staff) + `requireLogin()`/contact perms (client) |
| Templates | Smarty `.tpl` | Blesta `.pdt` |

**Key relief:** WHMCS addons hand-roll a dispatcher (`AdminDispatcher`/`ClientDispatcher` switching on
`$_REQUEST['action']`) because `_output`/`_clientarea` are single entry points. **Blesta plugin controllers
are directly routable** — every public method is reachable at `/…/plugin/{dir}/{controller}/{method}/`
(`create.md` §5). So you **delete the dispatcher** and turn each action into a controller method.

---

## 1. File & packaging mapping

| WHMCS addon | Blesta plugin | Notes |
|---|---|---|
| `modules/addons/{name}/{name}.php` (procedural) | `{name}/{name}_plugin.php` (class `NamePlugin extends Plugin`) | The `_*` functions become handler methods + controllers |
| `whmcs.json` + `_config()` name/description/author/version | `config.json` (`name`/`description` = **lang keys**, `authors`, `version`, `icon`) | `create.md` §2 |
| `_config()` `fields[]` (admin settings) | a settings form you build (Manage page / settings controller) — **not** `config.json` | §4 |
| `lib/Admin/AdminDispatcher.php` + `lib/Admin/Controller.php` | `controllers/admin_main.php` (class `AdminMain`) | Drop the dispatcher; actions → methods |
| `lib/Client/ClientDispatcher.php` + `lib/Client/Controller.php` | `controllers/client_main.php` (class `ClientMain`) | Same |
| `templates/*.tpl` (Smarty) | `views/default/*.pdt` | Re-author; admin = BS5/Paradigm, client = BS4 (§9) |
| `templates/js/*.js` | `views/default/javascript/*.js` (ship inside the plugin) | No core Gulp |
| `lang/english.php` | `language/en_us/{name}_plugin.php` (+ one per controller) | `create.md` §16 |
| `hooks.php` / `add_hook()` | `getEvents()` on the handler | §7 |
| custom tables (`Capsule::schema`) | `install()` via `Record` (`create.md` §4) | §3 |
| `composer.json` | `composer.json` `type: blesta-plugin` | |

---

## 2. `$vars` → Blesta

WHMCS passes every addon function a `$vars` array of the module config + a few system keys. Re-source each:

| WHMCS `$vars` key | What it is | Blesta equivalent |
|---|---|---|
| `'Text Field Name'`, … (by the field **key**) | your `_config()` `fields[]` values | your **own** settings storage (company setting or a plugin settings table) — §4 |
| `version` | installed module version | `$this->getVersion()` / the plugin's stored version |
| `modulelink` | admin entry URL (`addonmodules.php?module=…`) | your controller URL (`/admin/plugin/{dir}/admin_main/`) |
| `_lang` | loaded language array | `Language::loadLang(...)` + `$this->_('Key')` / `Language::_('Key', true)` |
| `$_REQUEST['action']` (dispatcher) | which sub-action | the **method name** in the URL (no manual dispatch) |

---

## 3. Function → Blesta mapping

| WHMCS addon function | Blesta plugin | Notes |
|---|---|---|
| `_config()` (name/description/author/language/version) | `config.json` metadata | literals → lang keys (`create.md` §2) |
| `_config()` `fields[]` (admin settings) | a settings form (Manage page §4) + storage | No auto-rendered config-fields system in Blesta |
| `_activate()` (create tables; returns status/description) | `install($plugin_id)` | Create tables via `Record`; **`$this->Input->setErrors()` to abort** (not a returned `status => 'error'`) |
| `_deactivate()` (drop tables) | `uninstall($plugin_id, $last_instance)` | Drop shared tables only when `$last_instance` |
| `_upgrade($vars)` (`$vars['version']` = installed) | `upgrade($current_version, $plugin_id)` | Branch on `$current_version` |
| `_output($vars)` (admin HTML, dispatcher) | **admin controller** (`AdminMain`) + a `nav_staff` action | §5 — directly routable; drop the dispatcher |
| `_sidebar($vars)` (admin sidebar HTML) | a **staff widget** action (`widget_staff_*`) or a card | §5 — no 1:1 "admin sidebar"; closest is a widget/card |
| `_clientarea($vars)` (returns page array) | **client controller** (`ClientMain`) + a `nav_client` action | §6 |
| `hooks.php` / `add_hook()` | `getEvents()` | §7 |
| admin access control / `requirelogin` | `getPermissions()`/`getPermissionGroups()` (staff) + `requireLogin()`/contact perms (client) | §8 |

---

## 4. Admin settings — `_config()` `fields[]` → a settings page

WHMCS auto-renders the `fields[]` from `_config()` on the addon's activation/config screen and feeds the
values back into every function via `$vars`. **Blesta has no equivalent auto-rendered config-fields
system** — a plugin builds and stores its own settings. Two idiomatic options:

- **The "Manage" page** (`controllers/admin_manage_plugin.php` → `AdminManagePlugin`) reached from
  *Settings > Company > Plugins > Manage* (`create.md` §9). Render the form, handle the POST, persist to
  company settings or your own settings table.
- **A dedicated settings controller/action** reached via a `nav_staff` action.

Store values with the `Companies` model settings or a plugin-owned table; read them where WHMCS read
`$vars['<field key>']`. The WHMCS field types (text/password/yesno/dropdown/radio/textarea) map to the same
`Form` helpers as everywhere (`reference/common/whmcs-mapping.md` has the type table).

> **WHMCS `_config()` `fields` is keyed by the field _key_, not the label.** `fields` is an associative
> array whose **key** is the config key; `FriendlyName` is separate *display* metadata inside each entry,
> and `$vars` exposes each value by that **field key**. The sample is misleading because it sets the key and
> `FriendlyName` to the same string (`'Text Field Name' => ['FriendlyName' => 'Text Field Name', …]`). When
> porting, key your Blesta settings by a **stable machine name** (e.g. `api_key`), not the human label.

---

## 5. Admin area — `_output` / `_sidebar`

`_output($vars)` is a single entry point that WHMCS calls for the addon's admin page; the sample switches on
`$_REQUEST['action']` via an `AdminDispatcher`. In Blesta:

- Make an **admin controller** `controllers/admin_main.php` (class `AdminMain extends AppController` or a
  plugin base), with `preAction()` calling `requireLogin()` and `setDefaultView('plugins'.DS.'{dir}'.DS)`.
- Each WHMCS dispatcher **action becomes a public method** (`index()`, `secret()`, …). **No manual
  dispatcher** — the router maps `/admin/plugin/{dir}/admin_main/{method}/` to the method (`create.md` §5).
- Register a **`nav_staff` action** in `getActions()` so the page appears in the admin menu (`create.md`
  §8). (WHMCS auto-lists addons under the Addons menu; Blesta needs the explicit nav action.)
- `echo`-style HTML output → **`$this->set('key', $value)`** and let Blesta **auto-render the `.pdt` that
  matches the action name** (e.g. `index()` → `views/default/admin_main.pdt`); a routed controller method
  normally returns nothing. Reserve **`$this->partial(...)`** for AJAX fragments and for the embedded
  **Manage** page (`AdminManagePlugin`), which must *return* its HTML (`create.md` §9). Admin = Paradigm/BS5
  (§9).

`_sidebar($vars)` (admin sidebar HTML) has **no exact Blesta equivalent**. Map it to a **staff widget**
(`widget_staff_home`/`widget_staff_client`/`widget_staff_billing` action) or a **card** (`getCards()` at
`level => 'staff'`) depending on where it should appear (`create.md` §8, §10).

---

## 6. Client area — `_clientarea` return array → a client controller

`_clientarea($vars)` returns a page-descriptor array. Map each key:

| `_clientarea()` return key | Blesta equivalent |
|---|---|
| `pagetitle` | set the page title (e.g. `$this->structure->set('title', …)` / view var) |
| `breadcrumb` | build breadcrumb markup/vars in your `.pdt` (no array contract) |
| `templatefile` (`'publicpage'`) | the `.pdt` you render (`client_main.pdt`) |
| `requirelogin` (bool) | `$this->requireLogin()` in `preAction()` when true; public action when false |
| `forcessl` | n/a (deprecated in WHMCS too) |
| `vars` (template variables) | `$this->set('key', $value)` for the view |

- Make `controllers/client_main.php` (class `ClientMain`), one **method per client action**
  (`index()`, `secret()`), reached at `/client/plugin/{dir}/client_main/{method}/`.
- Register a **`nav_client` action** (`getActions()`) for the client menu entry — and note this also makes
  the plugin gateable by **contact permissions** (`create.md` §13.1).
- `requirelogin => true` → `requireLogin()`; `requirelogin => false` → a public action (skip login in
  `preAction()`).

---

## 7. Hooks → `getEvents()`

A WHMCS addon's `hooks.php` (`add_hook('ClientEdit', 1, ...)`, navbar/sidebar hooks, etc.) becomes the
plugin handler's **`getEvents()`** plus, for nav/sidebar additions, **`getActions()`**/**`getCards()`**.
This is exactly the module-hooks story (`reference/modules/convert-from-whmcs.md` §9) — but for an addon the
events live in the **same plugin**, not a separate one:

| WHMCS hook | Blesta |
|---|---|
| service/client/invoice lifecycle hooks (`ClientEdit`, etc.) | `getEvents()` → `Clients.edit{Before,After}`, `Services.add{Before,After}`, … (`create.md` §11) |
| `ClientAreaPrimaryNavbar` add child | `getActions()` `nav_client` |
| `ClientAreaSecondarySidebar` panel | a card (`getCards()`) or client widget action |
| `AdminAreaPage`/admin hooks | `getActions()` `nav_staff` / staff widget |

Callbacks are `['this', 'methodName']` on the handler; the event object is an `EventInterface`
(`getParams()`/`setParams()`/`setReturnValue()`). Verify hook→event names against the core Observers
(`create.md` §11.3).

---

## 8. Access control

- **Admin:** WHMCS gates addon access by admin role on the config screen. Map to Blesta **staff ACL** —
  `getPermissionGroups()` + `getPermissions()` with `alias => '{dir}.admin_main'` and per-method `action`
  (`create.md` §13). You may attach to a core group instead of declaring your own.
- **Client:** `requirelogin => true` → `requireLogin()`. For per-**contact** restrictions, remember Blesta
  derives contact-permission options from your **`nav_client`/`widget_client_home` actions**, not from
  `level => 'client'` permission declarations (`create.md` §13.1) — register a client action if you want the
  plugin to be contact-gateable.

---

## 9. Templates & the admin/client theme split

| WHMCS | Blesta | Theme (6.0) |
|---|---|---|
| admin `.tpl` (from `_output`) | admin `.pdt` | **Paradigm / Bootstrap 5**, Bootstrap Icons, vanilla JS, light+dark (`reference/common/conversion-ui.md`) |
| client `.tpl` (from `_clientarea`) | client `.pdt` | **Client portal — Bootstrap 4 + jQuery + FontAwesome** (leave on BS4 for 6.0) |
| `templates/js/*.js` | `views/default/javascript/*.js` referenced directly | ship inside the plugin; no core Gulp |

Smarty → `.pdt` is a **rewrite**, not a syntax swap (`{$var}` → `<?php echo $var; ?>`, `{$LANG.x}` →
`$this->_('x')`, `{if}` → PHP). Convert **admin** views to Bootstrap 5; keep **client** views on Bootstrap 4
for 6.0.

---

## 10. Conversion checklist

- [ ] Addon rewritten as a **plugin**: `NamePlugin extends Plugin` handler + `controllers/` + `models/`;
      `composer.json` `type: blesta-plugin`.
- [ ] `_config()` metadata → `config.json` (lang-key `name`/`description`, `authors`, `version`,
      full-`bi bi-…` `icon`); `_config()` `fields[]` → a **settings page** (Manage page / settings
      controller) with your own storage (§4).
- [ ] `_activate` → `install()` (**`setErrors()` to abort**, not a `status => 'error'` return);
      `_deactivate` → `uninstall()` (drop shared tables only on `$last_instance`); `_upgrade` →
      `upgrade($current_version, $plugin_id)`.
- [ ] **Dispatchers deleted.** `_output`/`_clientarea` action-switches → one **public method per action** on
      `AdminMain`/`ClientMain` (directly routable); `nav_staff`/`nav_client` actions registered in
      `getActions()` for the menu entries.
- [ ] `_clientarea` return array mapped: `templatefile`→`.pdt`, `requirelogin`→`requireLogin()`,
      `vars`→`$this->set()`, `pagetitle`/`breadcrumb` set in the view.
- [ ] `_sidebar` → a staff widget action or card.
- [ ] `hooks.php`/`add_hook` → `getEvents()` on the handler (hook names → core Observer event names);
      nav/sidebar hooks → `getActions()`/`getCards()`.
- [ ] Access control: staff via `getPermissions()`/`getPermissionGroups()`; client via `requireLogin()`
      (+ a `nav_client` action if contact-gating is wanted).
- [ ] `lang/english.php` → `language/en_us/`; `.tpl` → `.pdt` (admin BS5/Paradigm, client BS4 + jQuery + FA);
      plugin JS/CSS shipped inside and referenced directly.

---

## 11. See also

- `create.md` — the full Blesta plugin contract (handler, routable controllers/models, actions/widgets,
  events, cron, ACL, cards, the Manage page, service tabs). **All mapping targets reference it.**
- `convert.md` — the 5.x→6.0 plugin port (UI/cosmetics) — complementary if also modernizing the UI.
- `reference/modules/convert-from-whmcs.md` — the module-side WHMCS port; shares the config-field-type,
  logging, and Smarty→`.pdt` patterns, and covers where a *provisioning/registrar* module's `hooks.php` goes.
- `reference/common/conversion-ui.md` — admin (Paradigm/BS5) view conversion for your admin `.pdt`.
- WHMCS `sample-addon-module` + `developers.whmcs.com/addon-modules/` — the WHMCS-side source of truth.
