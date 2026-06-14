# Authoring a Blesta 6 Module

> **Audience:** Developers (and AI tools) building a **new** Blesta 6.0 module from scratch.
> For porting an existing 5.x module, read `convert.md` first, then use this as the contract reference.
>
> **Source of truth:** the abstract base classes
> `components/modules/module.php` (`Module`) and
> `components/modules/registrar_module.php` (`RegistrarModule`),
> plus the official 6.0.0-b2 `cpanel` (hosting) and `namesilo` (registrar) modules.

---

## 0. The mental model

A **module** connects Blesta to an external system that provisions something a customer buys.
Blesta owns billing, orders, clients, and invoices; the module owns *"make it exist / change it / make it
go away on the remote system."*

Four layers of data flow through every module:

| Layer | What it is | Where it lives | Module method that produces it |
|---|---|---|---|
| **Module** | The installed extension (one per company that installs it) | `modules` table | `install()` returns nothing meaningful; metadata from `config.json` |
| **Module row** | A connection/account on the remote side (a server, a registrar account) | `module_rows` + `module_row_meta` | `addModuleRow()` / `editModuleRow()` return meta |
| **Package** | A sellable plan configured by the admin (links to a module row or group) | `packages` + `package_meta` | `addPackage()` / `editPackage()` return meta; `getPackageFields()` renders the form |
| **Service** | A single customer's provisioned instance of a package | `services` + `service_fields` | `addService()` etc. return service fields ("meta") |

Every "return meta" method returns a **numerically-indexed array** of:

```php
['key' => 'some_key', 'value' => 'some value', 'encrypted' => 0]  // encrypted => 1 to store encrypted
```

Returning `null` from a lifecycle method (suspend/cancel/renew/etc.) means *"leave existing service meta
unchanged."* Returning an array **replaces** the stored meta.

---

## 1. Choose the module flavour

```
Does your module register/transfer/renew domain names?
  ├─ YES → REGISTRAR module:  class extends RegistrarModule
  │                            config.json MUST set "type": "registrar"
  └─ NO  → HOSTING/SERVER (generic) module: class extends Module
                                config.json OMITS "type" (defaults to "generic")
```

`Module::getType()` returns `'generic'` when `config.json` has no `type` key — this is the correct,
intended value for hosting/server modules (cpanel ships with no `type`). **Do not invent a `"hosting"`
type.** `RegistrarModule::getType()` defaults to `'registrar'`.

The rest of this guide covers the generic contract first (§2–§14), then the registrar additions (§15).

---

## 2. Directory layout

```
your_module/
├── your_module.php          # main class: class YourModule extends Module
├── config.json              # metadata + behaviour declarations (see §3)
├── options.json             # OPTIONAL: importable configurable options (Blesta 6.1+; harmless in 6.0) — see §9.1
├── composer.json            # package metadata (type: blesta-module)
├── README.md
├── apis/                    # remote API client wrapper(s) — your code, your call
│   └── your_module_api.php
├── config/                  # optional: PHP config (email templates, field maps)
│   └── your_module.php
├── language/
│   └── en_us/
│       └── your_module.php  # $lang['YourModule.*'] strings
└── views/
    └── default/             # .pdt templates (Paradigm / Bootstrap 5)
        ├── images/logo.png
        ├── manage.pdt       # rendered by manageModule()
        ├── add_row.pdt      # rendered by manageAddRow()
        ├── edit_row.pdt     # rendered by manageEditRow()
        ├── admin_service_info.pdt
        ├── client_service_info.pdt
        └── tab_*.pdt        # one per service tab method
```

The class file name and class name follow Blesta's camel/snake convention: file `your_module.php` →
class `YourModule`. `Loader::fromCamelCase(get_class($this))` must round-trip to the directory name
(this is how `getEmailTemplate()` finds `config/your_module.php`).

---

## 3. `config.json`

Loaded in the constructor via `loadConfig()` and exposed as `$this->config` (a `stdClass`). Every metadata
getter (`getName`, `getVersion`, `getType`, `getAuthors`, `getLogo`, `getIcon`, `moduleRowName`, …) reads
from it. Full schema:

```json
{
    "version": "1.0.0",
    "name": "YourModule.name",
    "description": "YourModule.description",
    "authors": [
        { "name": "Your Company", "url": "https://example.com" }
    ],

    "service": { "name_key": "your_service_field" },
    "package": { "name_key": "your_package_field" },

    "module": {
        "row":     "YourModule.module_row",
        "rows":    "YourModule.module_row_plural",
        "group":   "YourModule.module_group",
        "row_key": "server_name"
    },

    "email_tags": {
        "module":  ["host_name"],
        "package": ["package"],
        "service": ["username", "password"]
    },

    "icon": "bi-server"
}
```

| Key | Meaning | Notes |
|---|---|---|
| `version` | Module version | Drives `upgrade($current_version)` |
| `name` / `description` | **Language keys**, not literals | Run through `Language::_()` by the getters |
| `authors[]` | name + url | |
| `service.name_key` | Which **service field** key labels a service in lists | `getServiceName()` reads it |
| `package.name_key` | Which **package field** identifies an as-yet-uncreated service | `getPackageServiceName()` reads it |
| `module.row` / `rows` | Singular/plural noun for a module row ("Server"/"Servers") | Language keys |
| `module.group` | Noun for a module group ("Server Group") | Language key |
| `module.row_key` | Which **row meta** key uniquely identifies a row | e.g. `server_name`, `user` |
| `email_tags` | Meta keys exposed as `{module.x}` / `{package.x}` / `{service.x}` in emails | Arrays of meta keys |
| `icon` | Bootstrap Icon class shown in Paradigm | e.g. `bi-server`, `bi-globe` |
| `type` | **OMIT for hosting/generic.** Set `"registrar"` for registrar modules | See §15 |
| `features` | **Registrar only** — gates `supports*` methods | See §15 |
| `logo` | Relative path to logo | Defaults to `views/default/images/logo.png` |

Two row-meta keys drive automatic capacity limiting (round-robin / first-available group ordering):
`account_limit` (the cap) and `account_count` (the current total). These default names are
`$module_row_field_limit` / `$module_row_field_total` on the base class — override the properties if your
row meta uses different keys.

> **Icon format — two different conventions, don't mix them up:**
>
> - **`config.json` `"icon"`** is the icon name **only**, e.g. `"bi-server"` / `"bi-globe"` (matching
>   the shipped cpanel/namesilo). Paradigm renders it as `<i class="bi <?php echo $module->icon;?>">`, i.e.
>   it **prepends the `bi` base class for you**. Writing `"bi bi-server"` here produces the broken
>   `class="bi bi bi-server"`.
> - **Everywhere else** — tab/link `'icon'` array values and inline `<i>` tags in views — use the **full**
>   class string `bi bi-server`, because nothing prepends `bi` in those contexts.

---

## 4. The class skeleton & constructor

```php
<?php
class YourModule extends Module   // or "extends RegistrarModule" for registrars
{
    public function __construct()
    {
        // 1. Load config.json into $this->config
        $this->loadConfig(dirname(__FILE__) . DS . 'config.json');

        // 2. Load components you need everywhere (Input is required for validation/errors)
        Loader::loadComponents($this, ['Input']);
        // Registrars typically also load Record (DB) and the PluginManager model:
        // Loader::loadModels($this, ['PluginManager']);
        // Loader::loadComponents($this, ['Input', 'Record']);

        // 3. Load this module's language pack
        Language::loadLang('your_module', null, dirname(__FILE__) . DS . 'language' . DS);

        // 4. Optional: load a PHP config file (email templates, field maps)
        // Configure::load('your_module', dirname(__FILE__) . DS . 'config' . DS);
    }
}
```

**Loading more on demand** inside methods (don't over-load in the constructor):

```php
Loader::loadModels($this, ['Clients', 'Packages', 'Services']);   // app models
Loader::loadComponents($this, ['Net', 'Record']);                 // components
Loader::loadHelpers($this, ['Html', 'Form', 'Date']);             // view helpers
```

`$this` exposes a lot via the `Container` trait. Useful inherited members:
`$this->getModule()`, `$this->getModuleRow($id=null)`, `$this->getModuleRows($group_id=null)`,
`$this->base_uri`, `$this->Input`.

---

## 5. Metadata methods (usually inherited)

You rarely override these — the base class derives them from `config.json`:
`getName()`, `getDescription()`, `getVersion()`, `getType()`, `getAuthors()`, `getLogo()`, `getIcon()`,
`moduleRowName()`, `moduleRowNamePlural()`, `moduleGroupName()`, `moduleRowMetaKey()`,
`getServiceName($service)`, `getPackageServiceName($packages, $vars)`.

---

## 6. Install / upgrade / uninstall lifecycle

```php
public function install() {}                       // set Input errors to abort install
public function upgrade($current_version) {}       // migrate when version changes
public function uninstall($module_id, $last_instance) {}  // $last_instance: last across all companies
public function cron($key) { return []; }          // run a cron task registered by this module
```

All are no-ops by default. Use `install()` to create custom tables (load `Record`), `upgrade()` to alter
them, and `uninstall()` to drop them when `$last_instance` is true. Set Input errors to prevent the action.

---

## 7. Module rows (connections to the remote system)

A module row is one server/account. The admin manages them under *Settings > Company > Modules*.

**Render the forms** (return HTML; load `Html`/`Form` helpers and `$this->view`):

```php
public function manageModule($module, array &$vars)  { /* return HTML for the module overview */ }
public function manageAddRow(array &$vars)            { /* return HTML for the "add server" form */ }
public function manageEditRow($module_row, array &$vars) { /* return HTML for the "edit server" form */ }
```

Typical `manageAddRow` body:

```php
public function manageAddRow(array &$vars)
{
    $this->view = new View('add_row', 'default');
    $this->view->base_uri = $this->base_uri;
    $this->view->setDefaultView('components' . DS . 'modules' . DS . 'your_module' . DS);
    Loader::loadHelpers($this->view, ['Form', 'Html', 'Widget']);
    $this->view->set('vars', (object)$vars);
    return $this->view->fetch();
}
```

**Persist the rows** (validate, then return row meta):

```php
public function addModuleRow(array &$vars)  { /* validate; return [['key'=>,'value'=>,'encrypted'=>], ...] */ }
public function editModuleRow($module_row, array &$vars) { /* same */ }
public function deleteModuleRow($module_row) {}
```

The base class provides naive default implementations that turn every `$vars` entry into an unencrypted
meta field — override to validate and to mark secrets `encrypted => 1`.

**Group ordering** (when a package targets a *group* of rows rather than one row):

```php
public function getGroupOrderOptions()
{
    return [
        'roundrobin' => Language::_('YourModule.order_options.roundrobin', true),
        'first'      => Language::_('YourModule.order_options.first', true)
    ];
}
// selectModuleRow($group_id) is implemented by the base class using FirstAvailable / RoundRobin
// schedules and the account_limit/account_count meta. Override only for custom selection logic.
```

---

## 8. Packages

```php
public function getPackageFields($vars = null)   { return new InputFields(); }  // render package form
public function addPackage(array $vars = null)    { /* validate; return package meta */ }
public function editPackage($package, array $vars = null) { /* validate; return package meta */ }
public function deletePackage($package) {}
```

`getPackageFields()` returns an `InputFields` object (see §11). The base `addPackage`/`editPackage`
default to copying `$vars['meta']` into meta fields — override to validate against the remote system (e.g.
fetch the list of plans from the server and present them as a `<select>`).

### 8.1 Modeling a server/control-panel package (SolusVM / VirtFusion / cPanel / Plesk style)

For provisioning modules, the **package meta** is where the admin pins down *what gets built*, and
`getPackageFields()` should populate selects by calling the remote API for the current row/group. Common
package-meta fields and how they're used:

| Meta concept | Typical key(s) | Source | Used in |
|---|---|---|---|
| Plan / package / product | `package`, `plan`, `plan_id` | API list for the selected row | `addService` (create with this plan), `changeServicePackage` |
| Location / node / group / cluster | `node`, `node_group`, `location_id`, `datacenter` | API list | `addService` placement |
| OS / template | `template`, `os_template`, `iso` | API list | `addService` |
| Resource defaults | `default_memory`, `default_disk`, `default_bandwidth`, `default_cpu` | admin input | `addService` defaults when no config option overrides |
| Networking | `dedicated_ip`, `ipv4`, `ipv6`, `extra_ips` | admin input / config option | `addService`, upgrades |
| Backups / extras | `backups`, `backup_plan` | admin input | `addService` |
| Account type | `type` (e.g. `standard`/`reseller`), `acl`, `account_limit` | admin input | `addService`, group limits |

**Config options override package defaults.** The package supplies the baseline; a configurable option of
the same concept (§9.1) overrides it per-order. Resolve with a clear precedence:

```php
$memory = $vars['configoptions']['memory'] ?? $package->meta->default_memory ?? null;
$disk   = $vars['configoptions']['disk']   ?? $package->meta->default_disk   ?? null;
$ips    = (int)($vars['configoptions']['extra_ips'] ?? 0);
```

**Package changes / upgrades** (`changeServicePackage($package_from, $package_to, $service, …)`): compare
the two packages' meta and the new config-option selections, then issue the remote resize/move/replan call.
Many panel modules also re-apply config-option-driven resources here (and in `editService`) so an upgrade
that only changes a config option (e.g. more RAM) still takes effect even when the package itself is
unchanged. Set Input errors to block a change the remote system rejects.

---

## 9. Service lifecycle — the heart of a module

These methods talk to the remote system. Signatures (defaults shown):

```php
public function validateService($package, array $vars = null) { return true; }
public function validateServiceEdit($service, array $vars = null) { return true; }

public function addService($package, array $vars = null, $parent_package = null,
                           $parent_service = null, $status = 'pending') { return []; }
public function editService($package, $service, array $vars = [],
                            $parent_package = null, $parent_service = null) { return null; }

public function suspendService($package, $service, $parent_package = null, $parent_service = null) { return null; }
public function unsuspendService($package, $service, $parent_package = null, $parent_service = null) { return null; }
public function cancelService($package, $service, $parent_package = null, $parent_service = null) { return null; }
public function uncancelService($package, $service, array $vars = [],
                                $parent_package = null, $parent_service = null) { return false; }
public function renewService($package, $service, $parent_package = null, $parent_service = null) { return null; }
public function changeServicePackage($package_from, $package_to, $service,
                                     $parent_package = null, $parent_service = null) { return null; }
```

Key conventions:

- `$status` in `addService` is one of `active|canceled|pending|suspended`. Provision only when appropriate.
- The **`use_module`** convention: order forms/admin pass `$vars['use_module']` (`'true'`/`'false'`).
  When `'false'`, **skip the remote call** but still return the service meta (admin is recording an
  existing account). cPanel: `if ($vars['use_module'] == 'true') { ...createacct... }`.
- `$parent_package`/`$parent_service` are non-null for **addon** services.
- Always call `validateService()` and bail on `$this->Input->errors()` before provisioning.
- Mark secrets `encrypted => 1` (passwords, API keys).

**Canonical `addService` shape** (abridged from cpanel):

```php
public function addService($package, array $vars = null, $parent_package = null,
                           $parent_service = null, $status = 'pending')
{
    $row = $this->getModuleRow();
    if (!$row) {
        $this->Input->setErrors(['module_row' => ['missing' =>
            Language::_('YourModule.!error.module_row.missing', true)]]);
        return;
    }

    $api = $this->getApi($row->meta->host_name, $row->meta->user_name, $row->meta->key);

    // ... derive/generate fields from $vars + $package ...
    $params = $this->getFieldsFromInput((array)$vars, $package);

    $this->validateService($package, $vars);
    if ($this->Input->errors()) {
        return;
    }

    if ($vars['use_module'] == 'true') {
        // Log BEFORE the call, masking secrets
        $masked = $params; $masked['password'] = '***';
        $this->log($row->meta->host_name . '|createacct', serialize($masked), 'input', true);

        $result = $this->parseResponse($api->createacct($params));   // parseResponse sets Input errors on failure
        if ($this->Input->errors()) {
            return;
        }
        $this->updateAccountCount($row);   // <-- YOUR OWN helper (not a base method); see note below
    }

    return [
        ['key' => 'username', 'value' => $params['username'], 'encrypted' => 0],
        ['key' => 'password', 'value' => $params['password'], 'encrypted' => 1],
    ];
}
```

> **Two helpers in that sample are NOT on the base class — you implement them:**
>
> - **`parseResponse()`** — a per-module convention: inspect the API response, call
>   `$this->log(..., 'output', $success)`, and `$this->Input->setErrors(...)` on failure.
> - **`updateAccountCount($row)`** — keeps the row's `account_count` meta in sync so group capacity
>   limits work (§7). There is **no base `updateAccountCount()`** — copying this call verbatim without
>   defining it is a fatal error. cPanel implements it by re-writing the row meta via `ModuleManager`:
>
>   ```php
>   private function updateAccountCount($module_row)
>   {
>       Loader::loadModels($this, ['ModuleManager']);
>       $count = $this->getAccountCount(/* your API */);   // however you count remote accounts
>       if ($count === false) {
>           return;
>       }
>       $vars = $this->ModuleManager->getRowMeta($module_row->id);   // current meta as stdClass
>       if ($vars) {
>           $vars->account_count = $count;
>           $this->ModuleManager->editRow($module_row->id, (array)$vars);
>       }
>   }
>   ```
>
>   Omit this entirely if your module doesn't use group capacity limits.

### 9.1 Configurable options (`$vars['configoptions']`)

**Configurable options** are admin-defined add-ons attached to a package (e.g. *Dedicated IP*, *Memory*,
*Disk*, *Extra IPs*, *Operating System*). The admin creates them under
*Packages > Configurable Options*, groups them, and assigns a group to a package. The customer then picks
values at order time. Many modules don't use them at all; provisioning modules like SolusVM lean on them
heavily.

**How they reach your module.** Selected option values are passed into the service lifecycle methods inside
`$vars['configoptions']` — a key/value array **keyed by the configurable option's machine `name`** (not its
display label), with the value being the customer's selection. (The base-class docblocks describe this as
"key is the package option label" — that wording is loose; the real key is the option `name`.)

```php
// e.g. an option named "customextraip" and one named "memory":
$vars['configoptions']['customextraip']  // => '2'
$vars['configoptions']['memory']         // => '4096'
```

**Where you use them.** Read `$vars['configoptions']` in `addService()`, `editService()`, and
`changeServicePackage()` to alter what you provision or upgrade — allocate extra IPs, set RAM/disk, pick an
OS template, etc. Validate them in `validateService()` / `validateServiceEdit()` (you can target an option
in `$rules` with a key like `'configoptions[customextraip]'`). Treat the option `name` as the stable
contract between the admin-configured option and your code — labels are translatable and editable, names
are not.

```php
public function addService($package, array $vars = null, /* ... */)
{
    $options = (array)($vars['configoptions'] ?? []);
    $memory  = $options['memory']        ?? $package->meta->default_memory;
    $disk    = $options['disk']          ?? $package->meta->default_disk;
    $extraip = (int)($options['customextraip'] ?? 0);
    // ...build the provisioning request from these...
}
```

### 9.2 Shipping options with the module — `options.json` (forward-looking)

> **Provenance / trust note:** This subsection is **not** derived from the shipped 6.0.0-b2 source — **no
> core module in 6.0.0-b2 currently ships an `options.json`.** It reflects the maintainers' `options.json`
> format and the 6.1 roadmap. The file is plain data and is only acted on when an admin explicitly imports
> it, so adding it is safe in 6.0; just don't treat it as an established 6.0 module contract.

So admins don't have to hand-build every option in a form, a module that supports configurable options
**can ship an `options.json`** in its root directory. Admins manage these under
*Packages > Configurable Options*, where they can create option groups/options manually **or import a group
(with its options) in one step**:

- **Blesta 6.0** — the admin opens the *import config option group* modal and **drag-and-drops an
  `options.json` file** into it. This already works today, so a shipped `options.json` is useful in 6.0:
  point the admin at your module's file (or let them download it) and drop it in.
- **Blesta 6.1+** — the import modal additionally **lists installed modules that ship an `options.json`**,
  so the admin can pick your module from a list instead of locating and dropping the file manually. Same
  file, same format — 6.1 just adds the auto-discovery convenience.

Either way the file is plain data (ignored unless explicitly imported), so it's **harmless to include in
6.0** and pays off immediately via drag-and-drop import. Ship it whenever your module supports config
options.

Shape (cPanel example — a single "Dedicated IP" yes/no option):

```json
{
    "id": 1,
    "name": "cPanel Configurable Options",
    "description": "Configurable options for cPanel hosting accounts",
    "hidden": 0,
    "hide_options": 0,
    "options": [
        {
            "id": 1,
            "label": "Dedicated IP",
            "name": "dedicated_ip",
            "description": "Whether the cPanel account should be provisioned with a dedicated IP address",
            "type": "select",
            "addable": 1,
            "editable": 1,
            "hidden": 0,
            "disable_pricing": 1,
            "hide_on_invoice": 1,
            "values": [
                {
                    "id": 1, "name": "No", "value": "0", "default": 1, "status": "active", "order": 0,
                    "min": null, "max": null, "step": null, "image": null,
                    "pricing": [
                        {
                            "id": 1, "term": 1, "period": "month",
                            "price": "0.0000", "price_renews": "0.0000", "price_transfer": null,
                            "setup_fee": "0.0000", "cancel_fee": "0.0000", "currency": "USD", "precision": 2
                        }
                    ]
                },
                {
                    "id": 2, "name": "Yes", "value": "1", "default": 0, "status": "active", "order": 1,
                    "min": null, "max": null, "step": null, "image": null,
                    "pricing": [
                        {
                            "id": 2, "term": 1, "period": "month",
                            "price": "0.0000", "price_renews": "0.0000", "price_transfer": null,
                            "setup_fee": "0.0000", "cancel_fee": "0.0000", "currency": "USD", "precision": 2
                        }
                    ]
                }
            ]
        }
    ],
    "condition_sets": []
}
```

Key fields:

| Field | Meaning |
|---|---|
| top-level `name` / `description` | The option **group** name/description |
| `hidden` / `hide_options` | Whether the group / its options are hidden from clients |
| `options[]` | The configurable options in the group |
| `options[].name` | **The machine name** your module reads from `$vars['configoptions']` — must match your code (e.g. `dedicated_ip`) |
| `options[].label` | Display label (translatable; **not** the provisioning key) |
| `options[].type` | `select`, `radio`, `checkbox`, `quantity`, `text`, etc. |
| `options[].addable` / `editable` | Whether the customer can set it at order / on edit |
| `options[].disable_pricing` | `1` = the option carries no price of its own |
| `options[].hide_on_invoice` | Hide the line from invoices |
| `values[]` | Selectable values; each has `value` (what your module receives), `default`, `order`, optional `min`/`max`/`step` (for quantity), and a `pricing[]` array |
| `values[].pricing[]` | Per term/period/currency pricing (price, renew, transfer, setup/cancel fees, precision) |
| `condition_sets` | Option dependency rules (show option B only when A = x); empty array if unused |

Match `options[].name` and `values[].value` exactly to what your provisioning code expects in
`$vars['configoptions']` — that string is the contract.

---

## 10. Service input fields (the order/admin forms)

```php
public function getAdminAddFields($package, $vars = null)  { return new InputFields(); }
public function getClientAddFields($package, $vars = null) { return new InputFields(); }
public function getAdminEditFields($package, $vars = null) { return new InputFields(); }
```

These return `InputFields` describing what the admin/client fills in when ordering/editing a service
(e.g. desired domain, username). The client form usually shows fewer fields than the admin form.

---

## 11. The field builder API (`InputFields` / `ModuleFields`)

The base class returns `InputFields` (`Blesta\Core\Util\Input\Fields\InputFields`). Older modules
(cpanel) still use the legacy `ModuleFields` — **same fluent API**, interchangeable for these purposes.
Prefer `InputFields` in new code.

```php
$fields = new InputFields();

// A labelled text field:
$label = $fields->label(Language::_('YourModule.field.domain', true), 'domain_id');
$label->attach($fields->fieldText('domain', $vars->domain ?? null, ['id' => 'domain_id', 'class' => 'form-control']));
$fields->setField($label);

// A select:
$opt = $fields->label(Language::_('YourModule.field.plan', true), 'plan_id');
$opt->attach($fields->fieldSelect('meta[plan]', $plans, $vars->meta['plan'] ?? null, ['id' => 'plan_id']));
$fields->setField($opt);

// Radios/checkboxes attach their own option labels (see Module::arrayToInputFields()).
// Inline JS / markup:
$fields->setHtml('<script>/* vanilla JS only for Paradigm */</script>');
```

Helper methods: `label()`, `fieldText()`, `fieldPassword()`, `fieldHidden()`, `fieldTextarea()`,
`fieldSelect()`, `fieldMultiSelect()`, `fieldCheckbox()`, `fieldRadio()`, `tooltip()`, plus `attach()`
(chain a control onto a label) and `setField()` (register a field) and `setHtml()`.

**Shortcut:** `Module::arrayToInputFields($arr, $fields = null, $vars = null)` builds an `InputFields`
object from a declarative array (`['name' => ['label'=>, 'type'=>'text|select|checkbox|radio|hidden',
'options'=>, 'attributes'=>]]`). `arrayToModuleFields()` is a deprecated alias.

> **UI caveat:** the Blesta `Form`/field helpers do **not** auto-inject Bootstrap 5 classes, and some
> shipped modules still emit jQuery inside `setHtml()`. For Paradigm-correct markup (class names, vanilla
> JS, light/dark mode), follow `reference/common/conversion-ui.md`.

---

## 12. Service display & tabs

```php
public function getAdminServiceInfo($service, $package)  { return ''; }   // HTML in admin service view
public function getClientServiceInfo($service, $package) { return ''; }   // HTML in client service view
public function getClientManagementContent($package, $service) { return ''; } // extra HTML on client mgmt page

// Tabs (preferred, service-aware):
public function getAdminServiceTabs($service)  { /* return ['tabMethod' => 'Title', ...] */ }
// Client tab icons render in the 6.0 client portal (FontAwesome) — use 'fas fa-...', NOT Bootstrap Icons:
public function getClientServiceTabs($service) { /* return ['tabMethod' => ['name'=>'Title','icon'=>'fas fa-gear']] */ }

// Legacy package-aware variants (still called as fallback by the service-aware ones):
public function getAdminTabs($package)  { return []; }
public function getClientTabs($package) { return []; }
```

Each tab key is a **public method name** on your class. The host controller validates the method against
the tab list and then calls it like this (verified in `admin_clients.php::processModuleTab`):

```php
$content = $module->{$method}($package, $service, $this->get, $this->post, $this->files);
$this->setServiceTabMessages($module->errors(), $module->getMessages());   // errors+messages harvested AFTER
```

So a tab method's signature is `($package, $service, array $get = null, array $post = null, array $files = null)`,
it returns rendered HTML, and it surfaces feedback by **setting** errors/messages on `$this` (the
controller reads them via `errors()` / `getMessages()` afterward — you don't echo them). Tab array values
may add a custom `href`; for the `icon`, **admin** tabs use `bi bi-...` (Bootstrap Icons) and **client**
tabs use `fas fa-...` (FontAwesome, the 6.0 client portal) — see the icon note in §3 and the
admin-vs-client table in §12.

**A complete tab that handles its own POST:**

```php
public function tabStats($package, $service, array $get = null, array $post = null, array $files = null)
{
    $this->view = new View('tab_stats', 'default');
    Loader::loadHelpers($this->view, ['Form', 'Html']);

    $row    = $this->getModuleRow();                       // already set to this service's row by the controller
    $fields = $this->serviceFieldsToObject($service->fields);  // see §12a

    // Handle a submitted form on this tab
    if (!empty($post)) {
        $api = $this->getApi($row->meta->host_name, $row->meta->user_name, $row->meta->key);
        $this->log($row->meta->host_name . '|changepass', serialize(['user' => $fields->username]), 'input', true);
        $result = $this->parseResponse($api->passwd($fields->username, $post['new_password']));

        if ($this->Input->errors()) {
            // parseResponse already set Input errors; the controller shows them. Repopulate the form below.
        } else {
            $this->setMessage('success', Language::_('YourModule.!success.password_updated', true));
        }
    }

    $this->view->set('service_fields', $fields);
    $this->view->set('vars', (object)($post ?? []));
    $this->view->setDefaultView('components' . DS . 'modules' . DS . 'your_module' . DS);
    return $this->view->fetch();
}
```

> **CRITICAL — admin theme ≠ client theme (6.0).** A module renders views in **two different host
> themes**, and they are **not** the same framework:
>
> | Surface | Methods | Host theme (6.0) | Write views in… |
> |---|---|---|---|
> | **Admin** | `manageModule`, `manage*Row`, `getAdminServiceInfo`, `getAdminServiceTabs`/`getAdminTabs`, admin tab methods, `getAdminAddFields`/`getAdminEditFields` | **Paradigm** | **Bootstrap 5**, vanilla JS (no jQuery), **Bootstrap Icons** (`bi bi-…`), light **and** dark mode |
> | **Client** | `getClientServiceInfo`, `getClientServiceTabs`/`getClientTabs`, client tab methods, `getClientAddFields`, `getClientManagementContent` | **`bootstrap` client portal** — still **Bootstrap 4 + jQuery + FontAwesome** in 6.0 | **Bootstrap 4**, jQuery OK, **FontAwesome** (`fas fa-…`) — match the client theme |
>
> This is exactly why the shipped cpanel/namesilo **client** views (`tab_client_*.pdt`,
> `client_service_info.pdt`) still use `form-group`, `btn-block`, `float-right`, and `fas fa-…` — that is
> **correct** for the 6.0 client portal, not an oversight. Don't "convert" client views to BS5 for 6.0.
> (A Bootstrap 5 client theme is planned for 6.1; until it ships, client views stay BS4.) For **admin**
> views, follow `reference/common/conversion-ui.md` in full.

### 12a. Reading the data you're handed

The objects passed into your methods have predictable shapes. Read them defensively (`??`, `isset`).

**`$service->fields`** — a numerically-indexed array of `{key, value}` objects (the service meta you
returned from `addService`). It is **not** an associative array. Use the inherited protected helper
`serviceFieldsToObject()` to turn it into a readable object:

```php
$fields = $this->serviceFieldsToObject($service->fields);
$username = $fields->username ?? null;     // keys are the 'key' values you stored
$password = $fields->password ?? null;     // decrypted automatically when the service is loaded for you
```

**`$package->meta`** — a `stdClass` of the package meta you returned from `addPackage`/`editPackage`
(keys are your meta `key`s):

```php
$plan = $package->meta->package ?? null;        // e.g. cPanel: the remote plan name
$type = $package->meta->type ?? 'standard';
```

**`$module_row->meta`** (a.k.a. `$row->meta`) — a `stdClass` of row meta from `addModuleRow`/`editModuleRow`
(server/connection details; secrets you stored `encrypted => 1` are decrypted here):

```php
$row = $this->getModuleRow();           // current row, or getModuleRow($id) for a specific one
$host = $row->meta->host_name;
$key  = $row->meta->key;                 // decrypted
```

**`$vars` in add/edit service** — the raw submitted input (associative), including `$vars['configoptions']`
(§9.1), `$vars['use_module']` (§9), `$vars['client_id']`, and your `getAdmin/ClientAddFields` field names.

**`$service->options`** — config option selections attached to the service (each has `option_name`,
`option_value`, etc.) — useful when you need the option *as stored on the service* rather than from the
incoming `$vars`.

---

## 13. Talking to the remote system

- Put your HTTP/API client in `apis/your_module_api.php` and load it with
  `Loader::load(dirname(__FILE__) . DS . 'apis' . DS . 'your_module_api.php');`, then `new YourModuleApi(...)`.
- For simple calls, the base class offers `httpRequest($method, $url, $params)` (wraps the `Net` `Http`
  component; array `$params` are `http_build_query`'d).
- **Log every call** with the inherited helper:

```php
protected function log($url, $data = null, $direction = 'input', $success = false)
```

  `$url` is a label (convention: `"$host|$operation"`), `$data` is serialized payload, `$direction` is
  `'input'` (request) or `'output'` (response). Returns an 8-char group id linking related entries.
  **Mask secrets** before logging.

---

## 14. Errors, messages, email tags, language

**Validation/errors** (load `Input` in the constructor):

```php
$this->Input->setErrors(['field' => ['rule' => Language::_('YourModule.!error.field.rule', true)]]);
if ($this->Input->errors()) { return; }     // bail
// public errors() returns the Input errors to the host app
```

`getCommonError($type)` returns a ready-made error array for `'invalid' | 'unsupported' | 'general'`.

**Soft messages** (non-fatal notices shown to the admin):

```php
$this->setMessage('success'|'error'|'notice', $text);   // retrieved by getMessages()
```

**Email tags & templates:**

```php
public function getEmailTags()     // defaults to config.json email_tags
public function getEmailTemplate() // loads config/<module>.php => '<Module>.email_templates'
```

Define welcome/notification templates in `config/your_module.php` as
`Configure::set('YourModule.email_templates', [ 'en_us' => ['text' => '...', 'html' => '...'] ])`.

**Language conventions** (`language/en_us/your_module.php`):

```php
$lang['YourModule.name']        = 'Your Module';
$lang['YourModule.description'] = '...';
$lang['YourModule.module_row']  = 'Server';
$lang['YourModule.!error.host_name.valid'] = 'The hostname appears to be invalid.';  // !error. prefix
$lang['YourModule.field.domain'] = 'Domain';
$lang['YourModule.tab_stats']    = 'Statistics';
```

Error keys use the `!error.` infix. Tab/field/label keys are free-form but stay consistent.

---

## 15. Registrar modules (the `RegistrarModule` subtype)

Domain registrars extend `RegistrarModule` (which extends `Module`, so **everything above still
applies**) and add a domain-specific surface. NameSilo is the canonical example.

### 15.1 config.json deltas

```json
{
    "type": "registrar",
    "service": { "name_key": "domain" },
    "package": { "name_key": "domain" },
    "module":  { "row_key": "user" },
    "features": {
        "dns_management":   1,
        "email_forwarding": 1,
        "id_protection":    1,
        "epp_code":         1
    },
    "icon": "bi-globe"
}
```

The `features` block gates the `supports*` methods, which the host app uses to show/hide configurable
options and tabs. You usually **don't override** the `supports*` methods — the base class reads them from
`config.json`:

```php
supportsDnsManagement()   // features.dns_management
supportsEmailForwarding() // features.email_forwarding
supportsIdProtection()    // features.id_protection
supportsEppCode()         // features.epp_code
supportsFeature($name)    // dispatches to supports<Name>()
```

### 15.2 The domain method surface (override what your registrar supports)

All default to "unsupported" (set Input error) or a safe no-op. Implement the ones your API supports:

```php
checkAvailability($domain, $module_row_id = null)            // bool
checkTransferAvailability($domain, $module_row_id = null)    // bool (defaults to !checkAvailability)
bulkCheckAvailability($domains, $module_row_id = null)       // [domain => bool]
isValidTerm($tld, $term, $transfer = false)                 // bool (default: term <= 10)

getTlds($module_row_id = null)                               // ['.com', '.net', ...]
getTldPricing($module_row_id = null)                         // see shape below
getFilteredTldPricing($module_row_id = null, $filters = [])  // same shape, filtered by tlds/currencies/terms

registerDomain($domain, $module_row_id = null, array $vars = [])    // bool
transferDomain($domain, $module_row_id = null, array $vars = [])    // bool
renewDomain($domain, $module_row_id = null, array $vars = [])       // bool
restoreDomain($domain, $module_row_id = null, array $vars = [])     // bool

getDomainInfo($domain, $module_row_id = null)               // registrar-specific assoc array
getServiceDomain($service)                                  // extract domain from service->fields['domain']
getRegistrationDate($service, $format = 'Y-m-d H:i:s')      // overridden: query registry / domains plugin
getExpirationDate($service, $format = 'Y-m-d H:i:s')        // overridden: query registry / domains plugin

getDomainContacts($domain, $module_row_id = null)          // [ ['external_id','email','phone','first_name',
                                                            //    'last_name','address1','address2','city',
                                                            //    'state','zip','country'], ... ]
setDomainContacts($domain, array $vars = [], $module_row_id = null)

getDomainNameServers($domain, $module_row_id = null)       // [ ['url'=>, 'ips'=>[...] ], ... ]
setDomainNameservers($domain, $module_row_id = null, array $vars = [])  // $vars = [ns1, ns2, ...]
setNameserverIps(array $vars = [], $module_row_id = null)  // ['ns1' => [ip, ...], ...]

getDomainIsLocked($domain, $module_row_id = null)          // bool
lockDomain($domain, $module_row_id = null)                 // bool
unlockDomain($domain, $module_row_id = null)               // bool

updateEppCode($domain, $epp_code, $module_row_id = null, array $vars = [])  // bool
sendEppEmail($domain, $module_row_id = null)               // bool
resendTransferEmail($domain, $module_row_id = null)        // bool
```

**`getTldPricing` return shape** (memorise this):

```php
[
  '.com' => [
    'USD' => [
       1 => ['register' => 8.99,  'transfer' => 8.99,  'renew' => 8.99],
       2 => ['register' => 17.98, 'transfer' => 17.98, 'renew' => 17.98],
       // ... terms in years ...
    ],
    'EUR' => [ /* ... */ ],
  ],
  '.net' => [ /* ... */ ],
]
```

### 15.3 Wiring domains into the standard service lifecycle

The generic lifecycle methods (§9) are where registration actually happens. NameSilo's `addService()`
calls `registerDomain()` or `transferDomain()` depending on whether an EPP/auth code was supplied;
`renewService()` calls `renewDomain()`; `suspendService()`/`cancelService()` typically just **disable
auto-renewal** (you can't "suspend" a domain). The domain method surface in §15.2 is *also* called
directly by the Domains plugin and by the service-management tabs.

### 15.4 Domains-plugin integration

`getExpirationDate()` / `getRegistrationDate()` are overridden in the base `RegistrarModule` to read
locally-cached dates from the `domains_domains` table when the **domains** plugin is installed (it falls
back to `service->date_renews` / `date_added`). Override them in your module to query the registrar's API
directly (NameSilo calls `getDomainInfo` and reads `expires` / `created`).

### 15.5 Registrar structural extras

- An **`includes/`** directory is common for registrar helpers (NameSilo: `communication.php` for client
  notices). Hosting modules rarely need it.
- Registrar **`apis/`** is usually split into command classes (`commands/domains.php`,
  `domains_dns.php`, `domains_transfer.php`, …) wrapping the single HTTP client.
- TLD-specific required fields (e.g. `.us`, `.ca`) live in `config/<module>.php` and are merged into the
  add-service fields conditionally.
- Service tabs are typically declared via `getAdminServiceTabs()` / `getClientServiceTabs()` and pruned at
  runtime based on `features` (e.g. hide the DNS tab when `dns_management` is off for that service).

---

## 16. A minimal working module (generic)

```php
<?php
// your_module/your_module.php
class YourModule extends Module
{
    public function __construct()
    {
        $this->loadConfig(dirname(__FILE__) . DS . 'config.json');
        Loader::loadComponents($this, ['Input']);
        Language::loadLang('your_module', null, dirname(__FILE__) . DS . 'language' . DS);
    }

    public function manageModule($module, array &$vars)     { return ''; }
    public function manageAddRow(array &$vars)              { return ''; }
    public function manageEditRow($module_row, array &$vars) { return ''; }

    public function getAdminAddFields($package, $vars = null)  { return new InputFields(); }
    public function getClientAddFields($package, $vars = null) { return new InputFields(); }

    public function addService($package, array $vars = null, $parent_package = null,
                               $parent_service = null, $status = 'pending')
    {
        // talk to remote system here when $vars['use_module'] === 'true'
        return [
            ['key' => 'example_id', 'value' => '12345', 'encrypted' => 0],
        ];
    }

    public function suspendService($package, $service, $parent_package = null, $parent_service = null)   { return null; }
    public function unsuspendService($package, $service, $parent_package = null, $parent_service = null) { return null; }
    public function cancelService($package, $service, $parent_package = null, $parent_service = null)    { return null; }
}
```

```json
// your_module/config.json
{
    "version": "1.0.0",
    "name": "YourModule.name",
    "description": "YourModule.description",
    "authors": [{ "name": "You", "url": "https://example.com" }],
    "service": { "name_key": "example_id" },
    "module":  { "row": "YourModule.module_row", "rows": "YourModule.module_row_plural", "row_key": "server_name" },
    "icon": "bi-box"
}
```

```php
// your_module/language/en_us/your_module.php
<?php
$lang['YourModule.name'] = 'Your Module';
$lang['YourModule.description'] = 'Provision things with Your System.';
$lang['YourModule.module_row'] = 'Server';
$lang['YourModule.module_row_plural'] = 'Servers';
```

---

## 17. Author's checklist

- [ ] Correct base class & `type` (registrar ⇒ `RegistrarModule` + `"type":"registrar"`; otherwise omit `type`).
- [ ] `config.json` keys are **language keys**, and every key resolves in `language/en_us/<module>.php`.
- [ ] Constructor loads config, `Input`, and the language pack.
- [ ] Every remote call is logged (`$this->log`) with secrets masked, and errors set via `$this->Input->setErrors`.
- [ ] Service meta marks secrets `encrypted => 1`.
- [ ] `use_module === 'true'` gate respected in `addService` (and edit/cancel where relevant).
- [ ] Lifecycle methods return `null` to keep meta, or a meta array to replace it.
- [ ] Configurable options (if used) are read from `$vars['configoptions']` by **machine name** and validated.
      *Optional:* ship an `options.json` if you want importable options (forward-looking, §9.2) — if you do,
      its `name`/`value` strings must agree with the code.
- [ ] `account_count` kept in sync if you use group limits.
- [ ] **Admin** views are Paradigm/Bootstrap 5, vanilla JS, Bootstrap Icons, light+dark safe
      (`reference/common/conversion-ui.md`); **client** views target the 6.0 client theme (BS4 + jQuery + FontAwesome). See §12.
- [ ] Registrar: `features` block matches the `supports*`/tabs you actually implement; `getTldPricing`
      returns the exact `[tld][currency][year]['register'|'transfer'|'renew']` shape.
- [ ] `composer.json` `type` is `blesta-module`.

---

## 18. Canonical reference files (read these to confirm any detail)

| What | Path (in a 6.0 build) |
|---|---|
| `Module` abstract contract | `components/modules/module.php` |
| `RegistrarModule` abstract contract | `components/modules/registrar_module.php` |
| Hosting/server example | `components/modules/cpanel/` |
| Registrar example | `components/modules/namesilo/` |
| Universal/config-driven example | `components/modules/universal_module/` |
| UI conversion rules | `reference/common/conversion-ui.md` |
