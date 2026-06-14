# Authoring a Blesta 6 Plugin

> **Audience:** Developers (and AI tools) building a **new** Blesta 6.0 plugin from scratch.
> For porting an existing 5.x plugin, read `convert.md` first, then use this as the contract reference.
>
> **Source of truth:** the abstract base class `components/plugins/lib/plugin.php` (`Plugin`), the
> `PluginManager` model (`app/models/plugin_manager.php`), the `Actions`/`Navigation`/`CronTasks`/
> `Permissions` models, the route table `config/routes.php`, and the official 6.0 plugins
> `support_manager`, `order`, `feed_reader`, `client_cards`, `system_overview`, `webhooks`.

---

## 0. The mental model

A **plugin** is the most powerful Blesta extension. Where a *module* provisions one kind of service and a
*gateway* processes payments, a **plugin extends the application itself**. A single plugin can ship:

- **Its own pages** — full MVC controllers/models/views, directly routable under `/admin/plugin/...` and
  `/client/plugin/...`.
- **Navigation links & dashboard widgets** — via `getActions()` (the *Actions/Navigation* system).
- **ACL permissions & permission groups** — via `getPermissions()` / `getPermissionGroups()` so staff
  access can be gated per controller/method.
- **Event listeners** — via `getEvents()`, hooking core operations (a client is deleted, an invoice closes,
  a service is added, …) to react or mutate behaviour.
- **Cron tasks** — registered against the `CronTasks` model and executed through `cron($key)`.
- **Profile/dashboard cards** — via `getCards()` (the metric/HTML cards on client profiles & dashboards).
- **Messenger templates** — via `getMessageTemplates()` (SMS/messenger notifications, alongside emails).
- **Service tabs** — via `getAdminServiceTabs()`/`getClientServiceTabs()` to add tabs to a service screen.
- **Custom DB tables, settings, email templates** — created in `install()`.

The **handler class** (`YourNamePlugin extends Plugin`) is thin: it's mostly *declarations* (the `get*()`
arrays above) plus the install/upgrade/uninstall lifecycle. The real work lives in your **controllers and
models**, which are ordinary Blesta MVC classes that happen to live inside the plugin directory.

### How the pieces register

`PluginManager::add()` (install) and `::upgrade()` call your handler's declaration methods and persist them:

| Handler method | Consumed by | Stored in |
|---|---|---|
| `install($plugin_id)` | run first; set errors to abort | (your tables/settings) |
| `getActions()` | `PluginManager::addAction()` → `Actions`/`Navigation` | `actions`, `navigation_items` |
| `getCards()` | `PluginManager::addCard()` | `plugin_cards` |
| `getEvents()` | `PluginManager::addEvent()` | `plugin_events` |
| `getMessageTemplates()` | `PluginManager::addMessage()` → `MessageGroups`/`Messages` | `message_groups`, `messages` |
| `getPermissionGroups()` | `PluginManager::addPermissionGroup()` (added **before** permissions) | `permission_groups` |
| `getPermissions()` | `PluginManager::addPermission()` | `permissions` |

> **Every one of these declaration arrays is "replace, not merge."** On install **and** upgrade,
> `PluginManager` re-reads them and **overwrites** the existing registrations (the base-class docblocks say
> "overwrites all existing …"). So they must always return the **complete** current set — never a delta.
> Cron tasks are the exception: they are *not* declared via a `get*()` method on the base class; you
> register them yourself with the `CronTasks` model (§14).

---

## 1. Directory layout

```
your_plugin/
├── your_plugin_plugin.php          # MAIN HANDLER: class YourPluginPlugin extends Plugin
├── your_plugin_controller.php      # OPTIONAL base controller: class YourPluginController extends AppController
├── your_plugin_model.php           # OPTIONAL base model: class YourPluginModel extends AppModel
├── config.json                     # metadata (see §2)
├── composer.json                   # package metadata (type: blesta-plugin)
├── controllers/
│   ├── admin_main.php              # AdminMain — staff page (extends your base controller)
│   ├── admin_manage_plugin.php     # AdminManagePlugin — the "Manage" settings page (see §10)
│   └── client_main.php             # ClientMain — client page
├── models/
│   └── your_plugin_things.php      # YourPluginThings extends YourPluginModel / AppModel
├── components/                     # optional: plugin-private components
├── lib/                            # optional: plugin-private libraries/helpers
├── config/                         # optional: PHP config (email/message templates, field maps)
│   └── your_plugin.php
├── language/
│   └── en_us/
│       ├── your_plugin_plugin.php  # $lang['YourPluginPlugin.*'] — handler strings
│       ├── admin_main.php          # one lang file per controller (matches controller name)
│       └── client_main.php
└── views/
    └── default/                    # .pdt templates + plugin-owned assets
        ├── images/logo.png
        ├── css/your_plugin.css     # ship your OWN css/js here; reference directly from views
        ├── javascript/your_plugin.js
        ├── admin_main.pdt
        ├── client_main.pdt
        └── admin_manage_plugin.pdt
```

**Naming conventions (enforced by the loader):**

- Handler file `your_plugin_plugin.php` → class `YourPluginPlugin` (directory name + `_plugin` →
  CamelCase + `Plugin`). `Plugins::create()` loads `PLUGINDIR/{dir}/{dir}_plugin.php` and requires the
  class to subclass `Plugin`.
- Controller file `admin_main.php` → class `AdminMain`. Model file `your_plugin_things.php` → class
  `YourPluginThings`. (`Loader::toCamelCase()` / `fromCamelCase()` round-trip the names.)
- A **base controller** (`your_plugin_controller.php` → `YourPluginController extends AppController`) and
  **base model** (`your_plugin_model.php` → `YourPluginModel extends AppModel`) are conventional — every
  admin/client controller extends the base controller; every model extends the base model. They're where
  you load the language pack and config once.

---

## 2. `config.json`

Loaded in the constructor via `loadConfig()` into `$this->config` (a `stdClass`). Plugin config is **much
smaller than a module's** — there are no service/package/module blocks. The base `Plugin` getters only read
six keys:

```json
{
    "version": "1.0.0",
    "name": "YourPluginPlugin.name",
    "description": "YourPluginPlugin.description",
    "icon": "bi bi-puzzle",
    "logo": "views/default/images/logo.png",
    "authors": [
        { "name": "Your Company", "url": "https://example.com" }
    ]
}
```

| Key | Meaning | Notes |
|---|---|---|
| `version` | Plugin version (literal) | Drives `upgrade($current_version, $plugin_id)` when it differs from the installed value |
| `name` | **Language key** (translated by `getName()`) | e.g. `"YourPluginPlugin.name"` |
| `description` | **Language key** (translated by `getDescription()`) | Some shipped plugins put a literal English string here — allowed, but a lang key is preferred |
| `icon` | Bootstrap Icons class shown in the admin plugin list | **Full class string here** — e.g. `"bi bi-headset"`, `"bi bi-cart"`. `getIcon()` returns it verbatim (no `bi` is prepended), unlike a module's `config.json` icon |
| `logo` | Relative path to the logo | Defaults to `views/default/images/logo.png` |
| `authors[]` | name + url objects | |

> **Plugin icon ≠ module icon convention.** A *module* config.json icon is the name only (`bi-server`) and
> the theme prepends `bi`. A *plugin* config.json icon is read raw by `Plugin::getIcon()`, so write the
> **full** `bi bi-...` string (the shipped `support_manager` uses `"bi bi-headset"`, `order` uses
> `"bi bi-cart"`). Action/nav icons are normalised separately (§8). (Cards have **no** icon field — §10.)

Everything else a plugin does — events, cron, permissions, cards, actions — is declared in **PHP** (the
`get*()` methods), **not** in `config.json`.

---

## 3. The handler class skeleton & constructor

```php
<?php
class YourPluginPlugin extends Plugin
{
    public function __construct()
    {
        // Load this plugin's language pack (handler strings)
        Language::loadLang('your_plugin_plugin', null, dirname(__FILE__) . DS . 'language' . DS);

        // Load config.json into $this->config
        $this->loadConfig(dirname(__FILE__) . DS . 'config.json');

        // Load components you need in the handler (Input is needed to set install/upgrade errors)
        Loader::loadComponents($this, ['Input']);

        // Optional: load a PHP config file (message/email templates, etc.)
        // Configure::load('your_plugin', dirname(__FILE__) . DS . 'config' . DS);
    }
}
```

`$this->Input` powers `errors()` — the base `Plugin::errors()` returns `$this->Input->errors()` if an
`Input` instance is present, which is how `PluginManager` detects an install/upgrade failure and rolls back.
Set errors with `$this->Input->setErrors([...])`.

**Inherited metadata methods (don't override):** `getName()`, `getDescription()`, `getVersion()`,
`getIcon()`, `getLogo()`, `getAuthors()` — all read from `config.json`. Plus `getMessages()` /
`setMessage($type, $msg)` for soft feedback and `errors()` for validation errors.

---

## 4. Install / upgrade / uninstall lifecycle

```php
public function install($plugin_id) {}                       // create tables/settings/email templates; set Input errors to abort
public function upgrade($current_version, $plugin_id) {}      // migrate when config.json version > installed version
public function uninstall($plugin_id, $last_instance) {}      // drop tables when $last_instance is true; remove cron/email
public function cron($key) { return []; }                    // run a cron task (see §14)
```

- **`install($plugin_id)`** runs **before** the declaration arrays are persisted. Create your custom tables
  here (load `Record`), seed settings, and add email templates / cron tasks. **Set `$this->Input->setErrors()`
  to abort** — `PluginManager::add()` deletes the half-installed `plugins` row if `errors()` is non-empty.
- **`upgrade($current_version, $plugin_id)`** runs only when `config.json`'s `version` differs from the
  stored version. Branch on `$current_version` with `version_compare()` and apply incremental migrations.
  Permissions/permission-groups are **re-added by `PluginManager` automatically** around your `upgrade()`
  call; your job is data/schema migration (and re-registering cron tasks / email templates that changed).
- **`uninstall($plugin_id, $last_instance)`** — `$last_instance` is true only when this is the final
  install across **all** companies. Drop shared tables and delete global cron task *definitions* only when
  `$last_instance` is true; always remove this company's per-company artifacts (cron task **runs**, etc.).

**Creating tables in `install()`** (load `Record`):

```php
public function install($plugin_id)
{
    Loader::loadModels($this, ['CronTasks']);
    if (!isset($this->Record)) {
        Loader::loadComponents($this, ['Record']);
    }

    try {
        $this->Record
            ->setField('id', ['type' => 'int', 'size' => 10, 'unsigned' => true, 'auto_increment' => true])
            ->setField('company_id', ['type' => 'int', 'size' => 10, 'unsigned' => true])
            ->setField('name', ['type' => 'varchar', 'size' => 255])
            ->setKey(['id'], 'primary')
            ->create('your_plugin_things', true);
    } catch (Exception $e) {
        $this->Input->setErrors(['db' => ['create' => $e->getMessage()]]);
        return;
    }

    // Register cron tasks (see §14)
    $this->addCronTasks($this->getCronTasks());
}
```

---

## 5. Controllers & routing — your plugin's pages

This is what sets plugins apart from modules: **a plugin's controllers are directly routable by URL.**

### 5.1 The route table

From `config/routes.php`, every request beginning `widget/` or `plugin/` (optionally prefixed by the admin
or client location) is stripped of that prefix and dispatched to the named controller:

```php
Router::route('^' . $admin_loc  . '/(widget|plugin)/(.+)', '$2');   // /admin/plugin/...  /admin/widget/...
Router::route('^' . $client_loc . '/(widget|plugin)/(.+)', '$2');   // /client/plugin/... /client/widget/...
Router::route('^(widget|plugin)/(.+)', '$2');                       // bare /plugin/... /widget/...
```

So the URL convention is:

| URL | Dispatches to | Use for |
|---|---|---|
| `/admin/plugin/your_plugin/admin_main/action/` | `AdminMain::action()` | staff pages |
| `/client/plugin/your_plugin/client_main/action/` | `ClientMain::action()` | client portal pages |
| `/plugin/your_plugin/controller/action/` | `Controller::action()` | context-neutral / public |
| `/admin/widget/your_plugin/admin_main/` | `AdminMain::index()` rendered as a widget | dashboard widgets |

The controller segment is the **file name** (`admin_main` → class `AdminMain`); the next segment is the
**method**; remaining segments are positional params in `$this->get[0]`, `$this->get[1]`, …

> **This is the key difference from app controllers.** In the *main* app, adding a new public method to a
> core controller is **not** routable (it falls through to `index()`); you must route through `index()`
> with a POST/GET discriminator. **Plugin controllers do not have this limitation** — every public method on
> a plugin controller is directly reachable at `/…/plugin/{dir}/{controller}/{method}/`. (This is why
> `support_manager` can expose `admin_tickets/reply`, `client_tickets/open`, etc. directly.)

### 5.2 The base controller & `preAction()`

Plugin controllers extend `AppController` (or a plugin-specific base that itself extends it). The
conventional split mirrors the app: name controllers `admin_*` for staff and `client_*` for clients, and
require the right login in `preAction()`.

```php
<?php
// your_plugin/your_plugin_controller.php  — shared base for all this plugin's controllers
class YourPluginController extends AppController
{
    public function preAction()
    {
        parent::preAction();

        // Render this plugin's views from views/default/
        $this->view->setDefaultView('plugins' . DS . 'your_plugin' . DS);

        // Load the language pack + (optionally) the plugin config
        Language::loadLang('your_plugin_plugin', null, PLUGINDIR . 'your_plugin' . DS . 'language' . DS);
    }
}
```

```php
<?php
// your_plugin/controllers/admin_main.php
class AdminMain extends YourPluginController
{
    public function preAction()
    {
        parent::preAction();
        $this->requireLogin();                                   // staff login + ACL (see §12)
        $this->uses(['YourPlugin.YourPluginThings', 'Clients']); // load models (note Plugin.Model namespacing)
        Language::loadLang('admin_main', null, PLUGINDIR . 'your_plugin' . DS . 'language' . DS);
    }

    public function index()
    {
        $this->set('things', $this->YourPluginThings->getAll($this->company_id));
        // renders views/default/admin_main.pdt automatically (view name = action name)
    }
}
```

`requireLogin()`, `$this->uses()`, `$this->helpers()`, `$this->set()`, `$this->post`, `$this->get`,
`$this->files`, `$this->company_id`, `$this->base_uri`, `$this->redirect()`, `$this->flashMessage()`,
`$this->partial()`, and `$this->Session` are all inherited `AppController` machinery — a plugin controller
is an ordinary Blesta controller.

### 5.3 AJAX & JSON

Same as the app: an `X-Requested-With: XMLHttpRequest` request triggers AJAX widget wrapping; use
`$this->outputAsJson(...)` + `return false` to emit raw JSON, and include the CSRF token
(`$this->Form->getCsrfToken()`) on POSTs. Because plugin controller methods *are* routable, you can expose a
clean AJAX endpoint as its own method (unlike app controllers).

---

## 6. Models

Plugin models live in `models/`, extend `AppModel` (often via a plugin base model), and are loaded with the
**`PluginName.ModelClass`** namespace so the loader knows to look inside the plugin directory:

```php
// inside a plugin handler/controller/model:
Loader::loadModels($this, ['YourPlugin.YourPluginThings']);   // -> $this->YourPluginThings
// or, in a controller:
$this->uses(['YourPlugin.YourPluginThings', 'Clients']);      // mixes plugin + core models
```

`YourPlugin` is the CamelCase of the plugin directory (`support_manager` → `SupportManager`). The class file
is `models/your_plugin_things.php` defining `class YourPluginThings extends YourPluginModel` (or
`extends AppModel`). A base model is where you load the plugin's language/config once:

```php
<?php
// your_plugin/your_plugin_model.php
class YourPluginModel extends AppModel
{
    public function __construct()
    {
        parent::__construct();
        Language::loadLang('your_plugin_plugin', null, PLUGINDIR . 'your_plugin' . DS . 'language' . DS);
        Configure::load('your_plugin', PLUGINDIR . 'your_plugin' . DS . 'config' . DS);
    }
}
```

Models use the inherited `$this->Record` query builder, `$this->Input` for validation, `$this->setErrors()`,
and `$this->errors()` — exactly like core models.

---

## 7. Views (`.pdt`) and the admin/client theme split

Plugin views are `.pdt` templates under `views/default/`, rendered by `setDefaultView('plugins'.DS.'your_plugin'.DS)`
+ returning from the action (view name defaults to the action name) or `$this->partial('view', $vars)`.

**The same admin-vs-client framework split as modules applies (6.0):**

| Surface | Controllers/views | Host theme (6.0) | Write views in… |
|---|---|---|---|
| **Admin** | `admin_*` controllers, admin widgets, `admin_manage_plugin`, admin service tabs | **Paradigm** | **Bootstrap 5**, vanilla JS (no jQuery), **Bootstrap Icons** (`bi bi-…`), light **and** dark mode |
| **Client** | `client_*` controllers, client dashboard cards/widgets, client service tabs | **`bootstrap` client portal** | **Bootstrap 4**, jQuery OK, **FontAwesome** (`fas fa-…`) |

For **admin** views follow `reference/common/conversion-ui.md` in full (BS5/Paradigm, light+dark). For
**client** views target BS4 + jQuery + FontAwesome (a BS5 client theme arrives in 6.1). **Ship your own CSS/JS
inside the plugin** (`views/default/css`, `views/default/javascript`) and reference it directly from the
`.pdt` — plugin assets do **not** go through core Gulp.

---

## 8. `getActions()` — navigation links, widgets & account-action links

`getActions()` returns a numerically-indexed array; `PluginManager::addAction()` persists each into the
`actions` table and (when enabled) a `navigation_items` row. **Where** an action appears is controlled by
its `location` (new form) or `action` (legacy form — auto-mapped).

```php
public function getActions()
{
    return [
        [
            'location' => 'nav_staff',                          // WHERE it shows (table below)
            'url'      => 'plugin/your_plugin/admin_main/',     // controller path (no leading slash)
            'name'     => 'YourPluginPlugin.nav.main',          // language key or literal label
            'icon'     => 'bi bi-puzzle',                       // optional; defaults to bi-gear
            'options'  => ['sub' => [                           // optional nested sub-nav items
                ['url' => 'plugin/your_plugin/admin_main/settings/', 'name' => 'YourPluginPlugin.nav.settings']
            ]],
            'enabled'  => 1                                     // optional, default 1
        ],
        [
            'location' => 'widget_staff_home',                  // a dashboard widget
            'url'      => 'widget/your_plugin/admin_main/',
            'name'     => 'YourPluginPlugin.name'
        ]
    ];
}
```

**Valid `location` values** (from `Actions::$locations` + `language/en_us/actions.php`):

| `location` | Renders… |
|---|---|
| `nav_staff` | A link in the **staff** (admin) navigation menu |
| `nav_client` | A link in the **client** navigation menu |
| `nav_public` | A link in the **public** navigation menu |
| `widget_staff_home` | A widget on the **staff dashboard** |
| `widget_client_home` | A widget on the **client dashboard** |
| `widget_staff_client` | A widget on the **staff client-profile** page |
| `widget_staff_billing` | A widget on the **staff billing** page |
| `action_staff_client` | A link in the **"Account Actions"** section of the staff client-profile page |

**Legacy `action` form still works.** Older plugins return `'action' => 'nav_primary_staff'` /
`'nav_primary_client'` / `'nav_secondary_*'` and `'uri' => ...`; `Actions::mapOldFields()` converts
`action`→`location` (`nav_primary_staff`/`nav_secondary_staff` → `nav_staff`, the client equivalents →
`nav_client`) and `uri`→`url`. `'nav_primary_client'` is special-cased to register **twice** — once for
`nav_client` and once for `nav_public`. New plugins should prefer `location`/`url`.

**Icon handling:** `Actions::add()` strips a leading `bi ` prefix before storing (templates re-prepend it),
so `'bi bi-puzzle'` and `'bi-puzzle'` both end up correct. Default icon is `bi-gear`. On **upgrade**,
`PluginManager` preserves an admin-customised icon and a previously-set `enabled` flag, and only lets a new
plugin default replace the generic gear.

**Sub-nav / secondary items:** `options.sub` (and the legacy `options.secondary`) become child
`navigation_items` parented to the primary action's URL (`convertOldActionParams()`).

> **Actions are wiped and re-added on every install *and* upgrade.** Return the **full** action set each
> time. To remove one action specifically you can call `PluginManager::deleteActions($plugin_id, $url)`, but
> normally you just stop returning it.

---

## 9. The "Manage" settings page — `AdminManagePlugin`

Every plugin gets a **Manage** button in *Settings > Company > Plugins > Installed*. Clicking it runs
`AdminCompanyPlugins::manage()`, which **embeds** a controller from your plugin — by default
`controllers/admin_manage_plugin.php` (class `AdminManagePlugin`), action `index` (both overridable via
`?controller=` / `?action=` query params). The URL is:

```
/admin/settings/company/plugins/manage/{plugin_id}/
```

The embedding controller is **primed manually** (it is *not* bootstrapped like a routed controller), so it
has a few peculiarities — verified in `admin_company_plugins.php::manage()`:

- It is constructed directly and given `->get`, `->post`, `->files`, `->uri`, `->base_uri`, `->action`,
  `->plugin` (the plugin dir), and crucially **`->parent`** (the `AdminCompanyPlugins` instance).
- Its action's **return value** is captured and set as the page `content` — so the action must **return**
  its rendered HTML (use `$this->partial(...)`), not echo it.
- Because it isn't auto-bootstrapped, use **`$this->parent`** for things normally done in `preAction()`:
  `$this->parent->requireLogin()`, `$this->parent->structure->set('page_title', ...)`.

```php
<?php
class AdminManagePlugin extends AppController
{
    private function init()
    {
        $this->parent->requireLogin();
        Language::loadLang('admin_manage_plugin', null, PLUGINDIR . 'your_plugin' . DS . 'language' . DS);
        $this->uses(['Companies', 'YourPlugin.YourPluginThings']);
        $this->plugin_id = $this->get[0] ?? null;

        $this->parent->structure->set('page_title', Language::_('YourPlugin.manage.page_title', true));
    }

    public function index()
    {
        $this->init();
        $vars = ['plugin_id' => $this->plugin_id, 'things' => $this->YourPluginThings->getAll()];
        $this->view->setView(null, 'YourPlugin.default');
        return $this->partial('admin_manage_plugin', $vars);   // MUST return the HTML
    }
}
```

> **Keep the Manage page simple.** The core comment is explicit: complex admin features should be a full
> plugin page reached via a `nav_staff` action (§8), not crammed into the embedded Manage controller — and
> you can redirect from the Manage page to a plugin-owned URL.

---

## 10. `getCards()` — dashboard & client-profile cards

Cards are the small metric/HTML tiles on the client dashboard and the staff client-profile page.
`PluginManager::addCard()` persists each into `plugin_cards`.

```php
public function getCards()
{
    return [
        [
            'level'         => 'client',                          // 'client' | 'staff' (optional, default 'client')
            'callback'      => ['this', 'getOpenTicketsCount'],   // [object|'this'|'Class', 'method']; serialized
            'callback_type' => 'value',                           // 'value' | 'html' (optional, default 'value')
            'label'         => 'YourPluginPlugin.card.tickets',   // REQUIRED — string/lang key under the value
            'link'          => 'plugin/your_plugin/client_tickets/',
            'text_color'    => '#fffaeb',                         // optional hex
            'background'    => '#007bff',                         // optional hex OR image path
            'background_type' => 'color',                         // 'color' | 'image' (optional, default 'color')
            'enabled'       => 1                                  // optional, default 1
        ]
    ];
}

// 'value' callback — return a scalar (number/string) shown big on the card:
public function getOpenTicketsCount($client_id)
{
    Loader::loadModels($this, ['YourPlugin.YourPluginThings']);
    return $this->YourPluginThings->getOpenCount($client_id);
}

// 'html' callback — return a markup string rendered inside the card instead of a value:
public function renderSomeCard($client_id) { return '<div>…</div>'; }
```

**Required vs optional** (from `PluginManager::getCardRules()`): `callback` and `label` are required;
`level` ∈ {client, staff}, `callback_type` ∈ {value, html}, `background_type` ∈ {color, image} are
validated when set; `callback` is unique per `plugin_id`+`level`. **There is no `icon` field on a card** —
the persisted columns are exactly `level, callback, callback_type, label, link, text_color, background,
background_type, enabled` (`PluginManager::addCard()`); any other key is ignored.

### 10.1 How a card is rendered at runtime

The card loaders live on `AppController` — `adminCards()` (staff client-profile) and `clientCards()`
(client dashboard), each calling `PluginManager::getCards($company_id, 'staff'|'client', true)` and
rendering the `admin_clients_card` / `client_main_card` partials. For **every** card they:

1. **Invoke the callback with the client id**: `$card->value = call_user_func($callback, $client_id)` (the
   `'this'`→plugin-instance substitution is applied first, same as events §11). So the callback **must be a
   public method on your handler class** (when using `'this'`) and is always passed **one argument — the
   client id**, for both staff and client cards.
2. **Render by `callback_type`** (in the partial):
   - `value` (default): `$card->value` is **HTML-escaped** and shown as the big stat value, with `label`
     beneath it and the `background`/`background_type` styling applied (`.stat-card`).
   - `html`: `$card->value` is echoed **raw** (unescaped) inside a `<div>` — your callback owns the markup,
     and `label`/`background` are **not** applied. You must escape your own output.
3. **Build the anchor from `link`.** The persisted field is **`link`**, but the loader derives a `uri` from
   it for the opening `<a href>` (the partials read `$card->uri` to open and `$card->link` to close). The
   `link`→`uri` derivation in `app_controller.php` is **three separate `if`s, not an `if/elseif` chain**, so
   later branches overwrite earlier ones. Net effective behaviour:
   - starts with `/` → web-root-absolute: `WEBDIR . trim($link, '/')` (this is the only branch that wins
     distinctly, because it's the one case the final "relative" `if` excludes);
   - **anything else** (including a leading `http…`) → treated as **relative to the area** and prefixed with
     the admin route (staff cards) or client route (client cards): `…/{Route.admin|client}/{link}`.

   > **External `http…` card links do NOT work as-is.** The code *intends* `http…` to be an external link
   > (the first `if` sets `uri = link`), but because the "relative URL" check is a separate `if`
   > (`substr($link,0,1) !== '/'`) rather than an `elseif`, it runs next and **overwrites** the external URL
   > with `…/{Route.admin|client}/http://…` — a broken link. This is a latent core bug (verified at
   > `app_controller.php` ~2374–2388 and ~2442–2456). Until it's fixed, **use a route-relative `link`** (no
   > scheme, no leading slash) or a **web-root-absolute** one (leading `/`); don't rely on external `http`
   > card links.

   So a **client** card with `'link' => 'plugin/your_plugin/client_tickets/'` resolves to
   `…/client/plugin/your_plugin/client_tickets/` — give the **route-relative** path (no leading slash) and
   it lands in the right area automatically. The card id is `{plugin_dir}_{snake_case(callback_method)}`.

---

## 11. `getEvents()` — hooking the application

Events let your plugin react to (and sometimes mutate) core operations. `getEvents()` returns
`['event' => <name>, 'callback' => <callable>]` pairs; `PluginManager::addEvent()` stores them (the callback
is **serialized**) and `triggerEvents()` invokes them whenever that event fires.

```php
public function getEvents()
{
    return [
        ['event' => 'Clients.delete',            'callback' => ['this', 'deleteClientThings']],
        ['event' => 'Invoices.setClosedAfter',   'callback' => ['this', 'onInvoiceClosed']],
        ['event' => 'Navigation.getSearchOptions','callback' => ['this', 'addSearchOption']],
    ];
}

public function onInvoiceClosed(\Blesta\Core\Util\Events\Common\EventInterface $event)
{
    $params = $event->getParams();           // associative array of event data
    // ...react. To pass data downstream / back to the caller:
    // $event->setReturnValue($value);
    // $event->setParams($modifiedParams);
}
```

### 11.1 The callback form & `'this'`

A callback is a PHP callable. The idiomatic plugin form is `['this', 'methodName']`: at trigger time
`PluginManager::invokePluginEvents()` **loads the plugin instance and substitutes the real object for the
`'this'` string**, then `call_user_func($callback, $event)`. A static `['Some\\Class', 'method']` works too
(but then the plugin must ensure that class is loadable). Each callback receives **one argument**: the
event object implementing `EventInterface`.

### 11.2 The `EventInterface` surface

From `core/Util/Events/Common/EventInterface.php`, a handler can read and mutate:

```php
$event->getName();                 // string event name
$event->getParams();               // array|null  — the data the trigger passed
$event->setParams(?array $params); // mutate params for later handlers
$event->getReturnValue();          // mixed       — accumulated return
$event->setReturnValue($value);    // set/replace the return value
$event->getErrors();  $event->setErrors(array $errors = []);
$event->getRules();   $event->setRules(array $rules = []);
```

When core code triggers an event via `AppModel::executeAndParseEvent($name, $params)`, the **return value**
and any param keys the handler echoes back are merged into the caller's result, and any **errors/rules** the
handler set are validated — so a `*Before` handler can veto or rewrite an operation, and an `*After` handler
can post-process it.

### 11.3 Which events exist

Event names follow `ClassName.method`. The core **Observers** (`core/Util/Events/Observers/`) and
**Handlers** (`core/Util/Events/Handlers/`) define them, almost always in **Before/After** pairs. A
representative (non-exhaustive) set:

```
Clients.add{Before,After}        Services.add{Before,After}        Invoices.add{Before,After}
Clients.edit{Before,After}       Services.edit{Before,After}       Invoices.edit{Before,After}
Clients.delete                   Services.cancel{Before,After}     Invoices.setClosed{Before,After}
Contacts.add{Before,After}       Services.suspend{Before,After}    Invoices.createFromServices{Before,After}
Transactions.add{Before,After}   Services.unsuspend{Before,After}
AppController.structure          AppController.preAction           Navigation.getSearchOptions
```

> **Discover the authoritative list at the source.** Grep the codebase for `Observer` method names and for
> `executeAndParseEvent(` / `triggerDeprecatedEvent(` calls — those strings are the exact event names you
> can hook. Some old names (e.g. `Invoices.setClosed`, `Services.add`) survive as **deprecated aliases** that
> still fire via `triggerDeprecatedEvent()`, so existing plugins hooking them keep working; prefer the
> `*After` form in new code.

The `webhooks` plugin is a useful pattern: it enumerates **every** available event and registers the same
`['this', 'listen']` callback for all of them.

---

## 12. Cron tasks

Plugins do **not** declare cron tasks via a base-class method. You register them with the **`CronTasks`
model** (in `install()`/`upgrade()`) and execute them in the handler's `cron($key)` method. There are two
records per task:

- **Task definition** (`cron_tasks`, global) — `CronTasks::add($vars)` with
  `['key', 'task_type' => 'plugin', 'dir' => '<plugin_dir>', 'name', 'description', 'is_lang', 'type' => 'time'|'interval']`.
- **Task run** (`cron_task_runs`, per-company schedule) — `CronTasks::addTaskRun($task_id, $vars)` with
  either `['interval' => <minutes>]` (for `type => 'interval'`) or `['time' => 'HH:MM:SS']` (for
  `type => 'time'`), plus `['enabled' => 1]`.

The shipped `order`/`support_manager`/`mass_mailer` plugins wrap this in a small `getCronTasks()` +
`addCronTasks()` helper pair (**their own** convention, not a base method) carrying a `type_value` that the
helper routes to `interval` or `time`:

```php
private function getCronTasks()
{
    return [
        // interval task — every 5 minutes
        ['key' => 'do_thing',     'task_type' => 'plugin', 'dir' => 'your_plugin',
         'name' => Language::_('YourPluginPlugin.cron.do_thing_name', true),
         'description' => Language::_('YourPluginPlugin.cron.do_thing_desc', true),
         'type' => 'interval', 'type_value' => 5,         'enabled' => 1],
        // time task — daily at 12:00 UTC
        ['key' => 'daily_report', 'task_type' => 'plugin', 'dir' => 'your_plugin',
         'name' => Language::_('YourPluginPlugin.cron.daily_report_name', true),
         'description' => Language::_('YourPluginPlugin.cron.daily_report_desc', true),
         'type' => 'time',     'type_value' => '12:00:00', 'enabled' => 1],
    ];
}

private function addCronTasks(array $tasks)
{
    Loader::loadModels($this, ['CronTasks']);
    foreach ($tasks as $task) {
        $task_id = $this->CronTasks->add($task);                       // add the global definition…
        if (!$task_id) {                                               // …or fetch it if it already exists
            $existing = $this->CronTasks->getByKey($task['key'], $task['dir'], $task['task_type']);
            $task_id = $existing->id ?? null;
        }
        if ($task_id) {
            $vars = ['enabled' => $task['enabled']];
            $task['type'] == 'interval'
                ? $vars['interval'] = $task['type_value']
                : $vars['time']     = $task['type_value'];
            $this->CronTasks->addTaskRun($task_id, $vars);             // per-company schedule
        }
    }
}

public function cron($key)
{
    switch ($key) {
        case 'do_thing':
            Loader::loadModels($this, ['YourPlugin.YourPluginThings']);
            $this->YourPluginThings->doThing();
            break;
        case 'daily_report':
            Loader::loadHelpers($this, ['Date']);
            // e.g. only on the 1st of the month: if ($this->Date->cast('c', 'j') == 1) { ... }
            break;
    }
    // return an array of log lines to record in the cron log (optional)
    return [];
}
```

**Removal in `uninstall()`** — delete this company's task run always; delete the global definition only on
the last instance:

```php
public function uninstall($plugin_id, $last_instance)
{
    Loader::loadModels($this, ['CronTasks']);
    foreach (['do_thing', 'daily_report'] as $key) {
        if (($run = $this->CronTasks->getTaskRunByKey($key, 'your_plugin', false, 'plugin'))) {
            $this->CronTasks->deleteTaskRun($run->task_run_id);
        }
        if ($last_instance && ($task = $this->CronTasks->getByKey($key, 'your_plugin', 'plugin'))) {
            $this->CronTasks->deleteTask($task->id, 'plugin', 'your_plugin');
        }
    }
}
```

Signatures (verified in `app/models/cron_tasks.php`): `add(array $vars)`,
`addTaskRun($task_id, array $vars)`, `getByKey($key, $plugin_dir = null, $task_type = 'plugin')`,
`getTaskRunByKey($key, $plugin_dir = null, $system = false, $task_type = 'plugin')`,
`deleteTask($task_id, $task_type, $dir)`, `deleteTaskRun($task_run_id)`. Times are **UTC**; intervals are
**integer minutes**.

---

## 13. ACL — `getPermissions()` & `getPermissionGroups()` (staff)

`getPermissions()` / `getPermissionGroups()` are the **staff** ACL — they gate which admin
controllers/methods a staff group can reach. **Groups must exist before permissions** (PluginManager adds
groups first). Both are re-added on install **and** upgrade (so return the full set).

```php
public function getPermissionGroups()
{
    // Only declare a group if you want a NEW staff-permissions SECTION of your own.
    // To slot permissions under an EXISTING Blesta section (Billing, Clients, …), omit this and
    // point group_alias at a core group instead (see below).
    return [
        [
            'name'  => Language::_('YourPluginPlugin.permission.main', true),
            'level' => 'staff',                       // 'staff' | 'client'
            'alias' => 'your_plugin.admin_main'       // ACO alias = plugin_dir.controller_name
        ]
    ];
}

public function getPermissions()
{
    return [
        [
            'group_alias' => 'your_plugin.admin_main',           // an EXISTING group — yours OR a core one
            'name'        => Language::_('YourPluginPlugin.permission.admin_main', true),
            'alias'       => 'your_plugin.admin_main',           // ACO alias → controller class AdminMain
            'action'      => '*'                                 // method (ACO action) or '*' for all
        ],
        [
            'group_alias' => 'your_plugin.admin_main',
            'name'        => Language::_('YourPluginPlugin.permission.admin_main_settings', true),
            'alias'       => 'your_plugin.admin_main',
            'action'      => 'settings'                          // gate only AdminMain::settings()
        ]
    ];
}
```

- **`alias`** = `{plugin_dir}.{controller_file_name}` → maps to the controller **class** (`AdminMain`).
- **`action`** = the controller **method** to gate, or `*` for the whole controller.
- **`group_alias` must match an *existing* permission group — not necessarily one your plugin declared.**
  `PluginManager::addPermission()` looks the group up by alias and **silently skips** the permission if no
  group is found, so you can attach permissions to a **core** Blesta group instead of inventing your own.
  The shipped `feed_reader` does exactly this: it declares **no** permission groups and attaches its
  permissions to the core `admin_billing` and `admin_main` groups
  (`plugins/feed_reader/feed_reader_plugin.php`). Declare your own group (and a section) only when the
  plugin warrants a distinct staff-permissions area.
- **Enforcement** happens through the standard staff ACL: a logged-in staff member's request to a gated
  controller/method is checked against the permissions granted to their staff group. At install,
  `PluginManager::add()` grants the installing staff group all of this plugin's permissions
  (`StaffGroups::grantPermission($staff_group_id, $plugin_id)`); admins manage the rest under
  *Staff > Groups*.

### 13.1 Client (contact) permissions are a *different* mechanism

> **Do not assume `getPermissionGroups(level => 'client')` restricts client-portal access.** Client-area
> restrictions for sub-account **contacts** are **not** driven by `getPermissions()`/`getPermissionGroups()`.
> They run through `Contacts::hasPermission($company_id, $contact_id, $area)`, which only knows about the
> options returned by `Contacts::getPermissionOptions()` — and that list is the hardcoded `client_*`
> built-ins **plus one `{plugin_dir}.*` option per plugin action registered at `nav_client` or
> `widget_client_home`** (verified in `app/models/contacts.php`). It does **not** read the
> `permissions`/`permission_groups` tables.

Consequences for a plugin that wants its **client** pages gateable per-contact:

- Register a **`nav_client`** (or `widget_client_home`) **action** (§8). That action's existence is what
  creates the `{plugin_dir}.*` contact-permission option an admin can then toggle for a contact.
- The client portal's `ClientController::requireLogin()` checks `{plugin_dir}.*` for the current contact.
- **If no such option exists, `Contacts::hasPermission()` returns `true` (unrestricted).** So a plugin that
  declares `level => 'client'` permission groups but ships **no** client action gives the *illusion* of
  contact restrictions while every contact actually has full access.

---

## 14. `getMessageTemplates()` — messenger notifications

Messages are the **messenger** (SMS/mobile/other) counterpart to email templates. `getMessageTemplates()`
returns templates that `PluginManager::addMessage()` installs into `message_groups`/`messages` for **every
installed language** of the company.

```php
public function getMessageTemplates()
{
    // Usually sourced from a PHP config file:
    Configure::load('your_plugin', dirname(__FILE__) . DS . 'config' . DS);
    return [
        [
            'action'  => 'YourPlugin.thing_updated',                  // unique trigger key 'Plugin.action'
            'type'    => 'staff',                                     // 'staff' | 'client' | 'shared'
            'tags'    => '{thing},{client},{staff.first_name}',       // comma-separated replacement tags
            'content' => [
                'sms' => "Thing #{thing.id} updated.\nStatus: {thing.status}"   // keyed by messenger type
            ]
        ]
    ];
}
```

| Field | Meaning |
|---|---|
| `action` | Unique message-group key (`Plugin.action`); also what you reference when sending |
| `type` | Audience: `staff`, `client`, or `shared` |
| `tags` | Comma-separated list of `{tag}` replacement variables available in the body |
| `content` | Map of **messenger type** → default template text (e.g. `sms`); installed per language |

**Sending** later is done through the messaging stack (the `Messages`/messenger models) keyed by your
`action`, supplying the tag values — mirroring how email templates are sent via the email service.
(Email templates, by contrast, are created directly in `install()` against the `Emails`/`EmailGroups`
models, not via `getMessageTemplates()`.)

---

## 15. Service tabs — `getAdminServiceTabs()` / `getClientServiceTabs()`

A plugin can add tabs to a **service** management screen (e.g. an AI assistant tab on every service). Opt in
with `allowsServiceTabs()` returning `true`, then declare tabs:

```php
public function allowsServiceTabs()
{
    return true;
}

public function getAdminServiceTabs(stdClass $service)
{
    return ['tabThing' => ['name' => Language::_('YourPlugin.tab_thing', true)]];   // admin: no icon needed
}

public function getClientServiceTabs(stdClass $service)
{
    return ['tabThing' => [
        'name' => Language::_('YourPlugin.tab_thing', true),
        'icon' => 'fas fa-robot'                                                     // client: FontAwesome
    ]];
}

// The tab method — key 'tabThing' => method tabThing(); returns rendered HTML:
public function tabThing(stdClass $service, array $get = null, array $post = null, array $files = null)
{
    $this->view = new View();
    $this->view->base_uri = $this->base_uri;
    $is_admin = (strpos($this->base_uri, '/admin/') !== false);
    $this->view->setView($is_admin ? 'tab_admin_thing' : 'tab_thing', 'YourPlugin.default');
    // load helpers/models, set vars, handle $post...
    return $this->view->fetch();
}
```

- The array key is the **method name** invoked to render the tab; the value is `name` (required) plus
  optional `icon` and `href`.
- **Icons:** admin tabs use Bootstrap Icons (`bi bi-…`) or none; **client** tabs use FontAwesome
  (`fas fa-…`) for 6.0 — same admin/client split as everywhere else.
- The tab method signature is `(stdClass $service, array $get = null, array $post = null, array $files = null)`
  and returns HTML. It can handle its own form POST and set messages/errors.

---

## 16. Errors, messages & language conventions

```php
// Validation errors (abort install/upgrade, or surface in a controller):
$this->Input->setErrors(['field' => ['rule' => Language::_('YourPluginPlugin.!error.field.rule', true)]]);
if ($this->Input->errors()) { return; }

// Soft, non-fatal feedback from the handler:
$this->setMessage('success'|'error'|'notice', $text);   // retrieved via getMessages()
```

**Language files** (`language/en_us/`): one file per "thing" — `your_plugin_plugin.php` for the handler,
plus a file per controller (`admin_main.php`, `client_main.php`) matching the controller name. Conventions:

```php
$lang['YourPluginPlugin.name']        = 'Your Plugin';
$lang['YourPluginPlugin.description']  = 'Does something useful.';
$lang['YourPluginPlugin.nav.main']     = 'Your Plugin';
$lang['YourPluginPlugin.card.tickets'] = 'Open Tickets';
$lang['YourPluginPlugin.permission.admin_main'] = 'Main';
$lang['YourPluginPlugin.!error.field.rule'] = 'That field is invalid.';   // !error. infix
```

Ship **`en_us` only** for a core-style plugin; the translation pipeline adds other locales.

---

## 17. A minimal working plugin

```php
<?php
// your_plugin/your_plugin_plugin.php
class YourPluginPlugin extends Plugin
{
    public function __construct()
    {
        Language::loadLang('your_plugin_plugin', null, dirname(__FILE__) . DS . 'language' . DS);
        $this->loadConfig(dirname(__FILE__) . DS . 'config.json');
        Loader::loadComponents($this, ['Input']);
    }

    public function install($plugin_id)
    {
        if (!isset($this->Record)) {
            Loader::loadComponents($this, ['Record']);
        }
        try {
            $this->Record
                ->setField('id', ['type' => 'int', 'size' => 10, 'unsigned' => true, 'auto_increment' => true])
                ->setField('company_id', ['type' => 'int', 'size' => 10, 'unsigned' => true])
                ->setKey(['id'], 'primary')
                ->create('your_plugin_things', true);
        } catch (Exception $e) {
            $this->Input->setErrors(['db' => ['create' => $e->getMessage()]]);
        }
    }

    public function uninstall($plugin_id, $last_instance)
    {
        if (!isset($this->Record)) {
            Loader::loadComponents($this, ['Record']);
        }
        if ($last_instance) {
            $this->Record->drop('your_plugin_things');
        }
    }

    public function getActions()
    {
        return [[
            'location' => 'nav_staff',
            'url'      => 'plugin/your_plugin/admin_main/',
            'name'     => 'YourPluginPlugin.nav.main',
            'icon'     => 'bi bi-puzzle'
        ]];
    }
}
```

```json
// your_plugin/config.json
{
    "version": "1.0.0",
    "name": "YourPluginPlugin.name",
    "description": "YourPluginPlugin.description",
    "icon": "bi bi-puzzle",
    "authors": [{ "name": "You", "url": "https://example.com" }]
}
```

```php
<?php
// your_plugin/controllers/admin_main.php
class AdminMain extends AppController
{
    public function preAction()
    {
        parent::preAction();
        $this->requireLogin();
        $this->view->setDefaultView('plugins' . DS . 'your_plugin' . DS);
        Language::loadLang('admin_main', null, PLUGINDIR . 'your_plugin' . DS . 'language' . DS);
        $this->uses(['YourPlugin.YourPluginThings']);
    }

    public function index()
    {
        $this->set('things', $this->YourPluginThings->getAll());
        // renders views/default/admin_main.pdt
    }
}
```

```php
// your_plugin/language/en_us/your_plugin_plugin.php
<?php
$lang['YourPluginPlugin.name'] = 'Your Plugin';
$lang['YourPluginPlugin.description'] = 'Does something useful.';
$lang['YourPluginPlugin.nav.main'] = 'Your Plugin';
```

---

## 18. Author's checklist

- [ ] Handler class is `YourPluginPlugin extends Plugin` in `your_plugin_plugin.php`; `composer.json`
      `type` is `blesta-plugin`.
- [ ] `config.json` `name`/`description` are **language keys** that resolve in
      `language/en_us/your_plugin_plugin.php`; `icon` is the **full** `bi bi-…` string.
- [ ] Constructor loads the language pack, `config.json`, and `Input`.
- [ ] `install()` creates tables and **sets `$this->Input->setErrors()` to abort** on failure; `uninstall()`
      drops shared tables only when `$last_instance`.
- [ ] Every `get*()` declaration returns the **complete** current set (they overwrite on install **and**
      upgrade).
- [ ] Actions use a valid `location` (or legacy `action`) and a `url` pointing at a real controller/method.
- [ ] Controllers extend `AppController` (or a plugin base), set the default view to
      `plugins/your_plugin/`, `requireLogin()` for staff pages, and load models via `Plugin.Model`
      namespacing.
- [ ] The Manage page (`AdminManagePlugin`) uses `$this->parent` for login/title and **returns** its HTML.
- [ ] Cron tasks registered via `CronTasks->add()` + `addTaskRun()`; `cron($key)` switches on the key;
      `uninstall()` removes runs (and definitions on last instance).
- [ ] Permission **groups** declared before permissions; `alias` = `dir.controller`, `action` = method/`*`.
- [ ] Event callbacks are public methods (use `['this','method']`); handlers read `$event->getParams()` and
      mutate via `setReturnValue()`/`setParams()`/`setErrors()` as needed; hooked event names are real
      (verified against the Observers).
- [ ] Cards' `callback`+`label` set; `'value'` callbacks return a scalar, `'html'` callbacks return markup.
- [ ] **Admin** views are Paradigm/Bootstrap 5, vanilla JS, Bootstrap Icons, light+dark safe
      (`reference/common/conversion-ui.md`); **client** views target BS4 + jQuery + FontAwesome.
- [ ] Plugin-owned CSS/JS ship inside the plugin and are referenced directly (no core Gulp).

---

## 19. Canonical reference files (read these to confirm any detail)

| What | Path (in a 6.0 build) |
|---|---|
| `Plugin` abstract contract | `components/plugins/lib/plugin.php` |
| Registration/install/upgrade engine | `app/models/plugin_manager.php` |
| Action locations & icon normalisation | `app/models/actions.php`, `language/en_us/actions.php` |
| Navigation items | `app/models/navigation.php` |
| Cron task/run API | `app/models/cron_tasks.php` |
| Permissions/ACL | `app/models/permissions.php`, `app/models/staff_groups.php` |
| Messenger templates | `app/models/message_groups.php`, `app/models/messages.php` |
| Event infrastructure | `core/Util/Events/` (`Common/EventInterface.php`, `Observers/`, `Handlers/`) |
| The "Manage" embedded-controller dispatcher | `app/controllers/admin_company_plugins.php::manage()` |
| Plugin/widget routing | `config/routes.php` |
| Rich reference plugin (pages, cron, events, perms, cards, messages, tabs) | `plugins/support_manager/` |
| Events + cards + cron + message templates | `plugins/order/` |
| Minimal widget-only plugin | `plugins/feed_reader/` |
| Cards-only plugin | `plugins/client_cards/` |
| "Listen to every event" pattern | `plugins/webhooks/` |
| UI conversion rules (admin) | `reference/common/conversion-ui.md` |
