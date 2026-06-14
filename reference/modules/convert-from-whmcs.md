# Porting a WHMCS Module to a Blesta 6 Module

> **Audience:** Developers (and AI tools) converting a **WHMCS** provisioning (server) or registrar module
> into a native **Blesta 6.0** module.
>
> **Read `create.md` first** — it is the Blesta-side contract (method surface, `config.json`, module rows,
> package meta, `InputFields`, service lifecycle, tabs). This file is the **WHMCS→Blesta mapping**: which
> WHMCS function becomes which Blesta method, and how the flat WHMCS `$params` array maps onto Blesta's
> structured objects.
>
> **Sources of truth:** the WHMCS SDK sample modules (`sample-provisioning-module`,
> `sample-registrar-module`) and the WHMCS developer docs
> (`developers.whmcs.com/provisioning-modules/`, `/domain-registrars/`). The Blesta side is the abstract
> `Module` / `RegistrarModule` base classes (`components/modules/`).
>
> **Shared WHMCS conventions** (licensing caveat, the re-source-don't-transform rule, config field-type
> table, error/logging translation, Smarty→`.pdt`, naming/packaging) live in
> **`reference/common/whmcs-mapping.md`** — read it once; this doc covers only what's module-specific.

---

## 0. The mental model — two different architectures

This is **not** a UI refresh like a 5.x→6.0 port; it's a **paradigm change**. You are rewriting, not
editing.

| | **WHMCS** | **Blesta** |
|---|---|---|
| Shape | **Procedural** — global functions prefixed with the module filename: `modulename_CreateAccount($params)` | **Object-oriented** — `class ModuleName extends Module { addService($package, $vars, …) }` |
| Dispatch | WHMCS calls `modulename_<Function>()` by naming convention | Blesta calls overridden methods on your class |
| Input | **One flat `$params` array** carrying everything (server creds, config options, client, service) | **Structured objects**: `$package`, `$service`, `$module_row`, plus a `$vars` array |
| Config UI | `_ConfigOptions()` returns positional `configoption1..24` | `getPackageFields()` returns `InputFields`; values stored as **named** package meta |
| Server binding | WHMCS "Servers" (global, in `tblservers`) | **Module rows** (`addModuleRow`/`getModuleRow`) |
| Success/failure | **return** the string `'success'` or an error message string | **`$this->Input->setErrors([...])`**; return service-meta array or `null` |
| Logging | `logModuleCall(module, func, request, response, …)` | `$this->log($url, $data, 'input'\|'output', $success)` |
| Client UI | `_ClientArea()` returns a **Smarty `.tpl`** + vars | `getClientServiceTabs()`/`getClientServiceInfo()` render **`.pdt`** |
| Hooks | `hooks.php` + `add_hook(...)` | **A Blesta plugin** (events) — *not* part of the module (see §11) |

**The single most important rule:** there is no mechanical 1:1 transform of the `$params` array. You must
**re-source each value** from the right Blesta object (§2). Treat the WHMCS function bodies as a *spec of
behaviour*, and re-express that behaviour against the Blesta contract.

---

## 1. File & packaging mapping

| WHMCS | Blesta | Notes |
|---|---|---|
| `modules/servers/{name}/{name}.php` | `{name}/{name}.php` (class `Name extends Module`) | Procedural file → one class |
| `modules/registrars/{name}/{name}.php` | `{name}/{name}.php` (class `Name extends RegistrarModule`) | + `"type":"registrar"` in `config.json` |
| `whmcs.json` (`type`, `name`, `category`, `description`, `logo`, `authors`) | `config.json` (`name`/`description` = **lang keys**, `authors`, `icon`, `module` block) + `composer.json` (`type: blesta-module`) | See `create.md` §3 |
| `_MetaData()` (`DisplayName`, `APIVersion`, ports, SSO labels) | `config.json` metadata + `getName()` etc. (mostly inherited) | Ports → see §5 |
| `logo.png` | `views/default/images/logo.png` | |
| `lib/` (API client) | `apis/` (your HTTP client) | `create.md` §13 |
| `templates/*.tpl` (Smarty) | `views/default/*.pdt` (Blesta template engine) | Re-author markup; see §8 |
| `hooks.php` / `add_hook()` | A separate **Blesta plugin** with `getEvents()` | §11 |

WHMCS module/file names are all-lowercase letters+numbers; Blesta's class file name must round-trip via
`Loader::fromCamelCase()` (file `your_module.php` ↔ class `YourModule`).

---

## 2. `$params` → Blesta data shapes (the core mapping)

WHMCS hands every function the same flat `$params`. In Blesta the equivalent values come from **different
objects**. This is the table to keep open while porting.

| WHMCS `$params` key | What it is | Blesta equivalent |
|---|---|---|
| `serviceid` | Service row id (`tblhosting.id`) | `$service->id` |
| `userid` | Owning client id | `$service->client_id` (and `$vars['client_id']` on add) |
| `domain` | Domain entered at order | `$service->fields` → your `domain` field via `serviceFieldsToObject()`; on add, `$vars['domain']` |
| `username` | Service username | a **service field** you stored (`serviceFieldsToObject($service->fields)->username`) |
| `password` | Service password | a service field stored **`encrypted => 1`** |
| `pid` | Product id | `$package->id` |
| `producttype` | `hostingaccount`/`reselleraccount`/`server`/`other` | (no direct equivalent — Blesta has no product-type enum; behaviour is per-module) |
| `configoption1..24` | **Positional** package config (from `_ConfigOptions()`) | **Named** `$package->meta->{yourkey}` (you choose the keys — §4) |
| `configoptions['Name']` | Configurable add-ons (by label) | `$vars['configoptions']['{machine_name}']` (by **name**, not label — `create.md` §9.1) |
| `customfields['Name']` | Product custom fields | a package/service field you model explicitly (no separate "custom fields" system in Blesta) |
| `serverid` | Assigned server | `$module_row->id` |
| `server` | bool: assigned to a server | whether `getModuleRow()` returns a row |
| `serverip` | Server IP | `$module_row->meta->host`/`ip` (your row meta key) |
| `serverhostname` | Server hostname | `$module_row->meta->host_name` (your key) |
| `serverusername` | Server API user | `$module_row->meta->user_name` |
| `serverpassword` | Server API password | `$module_row->meta->password` (stored `encrypted => 1`) |
| `serveraccesshash` | Server API key/hash | `$module_row->meta->key` (encrypted) |
| `serversecure` | bool: use SSL | `$module_row->meta->use_ssl` (your key) |
| `serverport` | Custom port | `$module_row->meta->port` |
| `clientsdetails` | Array of client profile (firstname, email, address…) | `Loader::loadModels($this,['Clients']); $this->Clients->get($service->client_id)` |
| `moduletype` | Module name | `get_class($this)` / `Loader::fromCamelCase(get_class($this))` |

> **`$params` carries decrypted secrets and server creds together; Blesta separates them.** In WHMCS the
> server password and the service password sit in the same array. In Blesta, server/connection secrets live
> in **module-row meta** (`$module_row->meta`, decrypted for you), and per-service secrets live in **service
> fields**. Read each from its proper place — do not try to reconstruct a single `$params`.

WHMCS reads `$service->fields` indirectly; in Blesta convert it once per method:

```php
$row    = $this->getModuleRow();                          // the "server"
$fields = $this->serviceFieldsToObject($service->fields); // username/password/etc you stored
$api    = $this->getApi($row->meta->host_name, $row->meta->user_name, $row->meta->key);
```

---

## 3. Provisioning functions → `Module` methods

| WHMCS function | Blesta `Module` method | Conversion notes |
|---|---|---|
| `_ConfigOptions()` *(required in WHMCS)* | `getPackageFields($vars)` → `InputFields`; persist via `addPackage`/`editPackage` meta | Positional → named; §4 |
| `_CreateAccount($params)` | `addService($package, $vars, $parent_package, $parent_service, $status)` | Return **service-meta array** (not `'success'`). Honour `$vars['use_module']==='true'` gate. `create.md` §9 |
| `_SuspendAccount($params)` | `suspendService($package, $service, …)` | return `null` to keep meta |
| `_UnsuspendAccount($params)` | `unsuspendService($package, $service, …)` | |
| `_TerminateAccount($params)` | `cancelService($package, $service, …)` | "Terminate" = Blesta **cancel** |
| `_ChangePassword($params)` | usually a **service tab** method (`getAdminServiceTabs`/`getClientServiceTabs` + `tabX()`), or `editService` | WHMCS exposes it as a built-in action; Blesta models it as a tab or admin action |
| `_ChangePackage($params)` | **split in two** — `changeServicePackage($package_from, $package_to, $service, …)` for a package/pricing change; `editService($package, $service, $vars, …)` for configurable-option changes | One WHMCS fn handles both; Blesta routes them separately — see note below |
| `_Renew($params)` | `renewService($package, $service, …)` | |
| `_TestConnection($params)` (returns `['success'=>,'error'=>]`) | a connection check inside `addModuleRow`/`editModuleRow` validation, or a helper invoked from `manageModule` | Blesta has no built-in "Test Connection" button contract; validate the row on save and set Input errors |
| `_AdminCustomButtonArray()` + handler fns | admin **service tabs** (`getAdminServiceTabs()` + `tabX()`) | Each WHMCS button → a tab/action method |
| `_ClientAreaCustomButtonArray()` + handler fns | client **service tabs** (`getClientServiceTabs()` + `tabX()`) | |
| `_AdminServicesTabFields($params)` | `getAdminServiceInfo($service, $package)` (display) and/or an admin tab | Return HTML; editable fields → handle POST in the tab method |
| `_AdminServicesTabFieldsSave($params)` | POST handling inside the same admin tab method (reads `$post`) | Blesta passes `$post` to the tab method — no separate "save" fn |
| `_ClientArea($params)` (returns tpl + vars) | `getClientManagementContent()` (default manage-page body) / `getClientServiceInfo()` / a client tab method returning rendered `.pdt` | §6 |
| `_ServiceSingleSignOn($params)` | a tab/action that builds and redirects to an SSO URL | No core SSO contract; implement as an action returning/redirecting to the URL |
| `_AdminSingleSignOn($params)` | a `manageModule`/row action that builds the admin SSO URL | |
| `_MetaData()` | `config.json` + inherited getters; ports → row-meta defaults (§5) | |

Methods with **no WHMCS equivalent** that you should still implement on the Blesta side:
`validateService()` (validate `$vars` before provisioning), `getAdminAddFields()`/`getClientAddFields()`
(the order/admin service form — WHMCS auto-generates these from product settings; in Blesta you build them),
`manageModule()`/`manageAddRow()`/`manageEditRow()` (the server admin UI), and `install()`/`uninstall()`
for any custom tables.

> **`_ChangePackage` splits into two Blesta methods — and `changeServicePackage()` has NO `$vars`.** WHMCS
> uses one function for both product up/downgrades and configurable-option changes. Blesta routes them
> separately (verified in `app/models/services.php`):
>
> - A **package/pricing change** calls `changeServicePackage($package_from, $package_to, $service,
>   $parent_package, $parent_service)` — note the signature has **no `$vars` parameter**
>   (`components/modules/module.php`). Diff the two **package objects'** meta; you can read the service's
>   currently-stored option selections from **`$service->options`**, but the *incoming* new selections are
>   **not** passed here.
> - A **configurable-option change** flows through **`editService($package, $service, $vars, …)`** (and
>   `validateServiceEdit($service, $vars)`), where the new selections arrive in **`$vars['configoptions']`**
>   (already name/value-formatted by the host). So put config-option-driven resize/reprovision logic in
>   `editService()`, not `changeServicePackage()`.

### 3.1 Error & logging translation (applies to every function)

```php
// WHMCS:
//   logModuleCall('provisioningmodule', __FUNCTION__, $params, $response, $e->getTraceAsString());
//   return $e->getMessage();          // or 'success'
//
// Blesta:
$this->log($row->meta->host_name . '|createacct', serialize($masked_request), 'input', true);
$response = $api->create($request);
$this->log($row->meta->host_name . '|createacct', serialize($response), 'output', $success);
if (!$success) {
    $this->Input->setErrors(['api' => ['create' => $message]]);   // do NOT return a string
    return;
}
return [ ['key' => 'username', 'value' => $username, 'encrypted' => 0], /* … */ ];
```

- WHMCS "return `'success'` or an error string" → Blesta "**set Input errors on failure**; return meta on
  success." The host app checks `->errors()`, not a return string.
- `logModuleCall(...)` → **two** `$this->log()` calls (request `'input'`, response `'output'`), **masking
  secrets** first. The label convention is `"$host|$operation"`.

---

## 4. Config options — the #1 gotcha

WHMCS conflates several concepts that Blesta keeps distinct. Map them deliberately:

| WHMCS concept | What it is | Blesta target |
|---|---|---|
| **Product Config Options** (`_ConfigOptions()`, `configoption1..24`) | Admin-set, **per-product** module settings (plan name, disk, toggles) | **Package meta** via `getPackageFields()` → `$package->meta->{key}` |
| **Configurable Options** (`configoptions['Label']`) | Customer-selectable **add-ons** priced separately | **Blesta Configurable Options** → `$vars['configoptions']['{name}']` (`create.md` §9.1); ship an `options.json` (§9.2) |
| **Custom Fields** (`customfields['Label']`) | Per-product extra data fields | Model explicitly as package/service fields — Blesta has no separate "custom fields" subsystem |

**The positional→named trap.** WHMCS code reads `$params['configoption1']`, `$params['configoption3']` —
the **index** is the contract, and reordering `_ConfigOptions()` silently breaks things. In Blesta you give
each a **name** in `getPackageFields()` and read `$package->meta->disk`, `$package->meta->enable_ftp`. When
porting, build a translation note:

```
WHMCS _ConfigOptions() order      ->  Blesta package-meta key
1  'Text Field'      (disk MB)    ->  meta[disk]
2  'Password Field'  (secret)     ->  meta[secret]      (encrypted => 1)
3  'Checkbox Field'  (yesno)      ->  meta[enable_ftp]  ('1'/'0')
4  'Dropdown Field'                ->  meta[plan]
5  'Radio Field'                   ->  meta[tier]
6  'Textarea Field'                ->  meta[notes]
```

WHMCS field types → Blesta `InputFields` builders (`create.md` §11):

| WHMCS `Type` | Blesta builder |
|---|---|
| `text` | `fieldText()` |
| `password` | `fieldPassword()` (+ store meta `encrypted => 1`) |
| `yesno` | `fieldCheckbox()` (value `'1'`/`'0'`) |
| `dropdown` | `fieldSelect()` (assoc `Options` → option array) |
| `radio` | `fieldRadio()` (CSV `Options` → option array) |
| `textarea` | `fieldTextarea()` (`Rows`/`Cols` → attributes) |

---

## 5. Servers → module rows

WHMCS "Servers" are global rows in `tblservers`; Blesta uses **module rows** owned by the module.

- Build the server form in **`manageAddRow()`/`manageEditRow()`** and persist in
  **`addModuleRow()`/`editModuleRow()`** (`create.md` §7). Map `serverhostname/username/password/accesshash/
  secure/port` → your row-meta keys, marking secrets `encrypted => 1`.
- `_MetaData()`'s `DefaultNonSSLPort` / `DefaultSSLPort` → defaults you pre-fill in the add-row form (and
  pick between based on your `use_ssl` row-meta).
- `_MetaData()`'s `RequiresServer` → if false, the product isn't server-bound; in Blesta that's a module
  that simply doesn't define/require a row (handle a null `getModuleRow()`).
- `_TestConnection()` → run the same check during `addModuleRow`/`editModuleRow` validation (call the API,
  `setErrors` on failure) and/or expose it from `manageModule()`. There is no built-in Test-Connection
  button contract in Blesta.
- For multi-server placement, WHMCS auto-selects; Blesta uses **module groups** + `getGroupOrderOptions()` /
  `selectModuleRow()` and the `account_limit`/`account_count` row meta (`create.md` §7).

---

## 6. Client area & admin display

| WHMCS | Blesta | Notes |
|---|---|---|
| `_ClientArea()` → `tabOverviewReplacementTemplate` | `getClientManagementContent($package, $service)` (default manage-page body) and/or a client tab method returning a full `.pdt` | `getClientManagementContent()` renders the **default tab** content of the client service manage page (`create.md` §12); use it for a replacement-style overview |
| `_ClientArea()` → `tabOverviewModuleOutputTemplate` | `getClientServiceInfo($service, $package)` (appended HTML) | Adds to the default overview |
| `_ClientAreaCustomButtonArray()` | additional client tabs | one tab per action |
| `_AdminServicesTabFields()` | `getAdminServiceInfo()` + admin tab for editable rows | |
| Smarty `.tpl` (`{$var}`, `{$LANG.x}`, `{if}`) | Blesta `.pdt` (`<?php echo $var; ?>`, `$this->_('Lang.x')`, plain PHP) | Re-author; **client portal is BS4 + jQuery + FontAwesome** in 6.0 |

> **Templating is a full rewrite, not a syntax swap.** WHMCS `.tpl` is Smarty; Blesta `.pdt` is its own
> PHP-based engine. Re-author the markup. And remember the **admin/client theme split** (`create.md` §12):
> admin views = Paradigm/Bootstrap 5/Bootstrap Icons/vanilla JS/light+dark (`reference/common/
> conversion-ui.md`); **client** views stay **Bootstrap 4 + jQuery + FontAwesome** for 6.0. The WHMCS
> sample `.tpl` uses BS-style `col-sm-*`/`btn` markup and `fa-` icons — for the **client** side that's
> roughly the right target (BS4); for any **admin** view, convert to BS5.

---

## 7. The service form WHMCS generates for you

WHMCS auto-builds the order/admin "configure service" form from product settings, custom fields, and config
options. Blesta does **not** — you must supply it explicitly:

- `getAdminAddFields($package, $vars)` / `getClientAddFields($package, $vars)` → the fields shown when an
  admin/client orders the service (e.g. desired domain, username).
- `getAdminEditFields($package, $vars)` → admin edit form.
- `validateService($package, $vars)` / `validateServiceEdit($service, $vars)` → validation that WHMCS would
  have done via field rules; set Input errors.

---

## 8. Single sign-on, custom buttons, sync

- **Custom buttons** (`_AdminCustomButtonArray` / `_ClientAreaCustomButtonArray`) — each entry becomes a
  **service tab** (or an admin action) whose method performs the action and returns HTML / sets messages.
- **SSO** (`_ServiceSingleSignOn` / `_AdminSingleSignOn`) — implement as a tab/action that builds the
  provider's login URL and redirects (or returns it). No core SSO method to override.
- **Usage/stats** shown in WHMCS `_ClientArea` → fetch in the tab method and render in the `.pdt`.

---

## 9. Hooks → a Blesta plugin (out of scope for the module)

WHMCS modules often ship a `hooks.php` registering `add_hook('ClientEdit', …)`,
`add_hook('ClientAreaPrimaryNavbar', …)`, sidebar panels, etc. **Blesta modules cannot register events.**
Those belong in a **Blesta plugin** (`getEvents()`, `getActions()` for nav, cards for sidebar-style panels)
— see `reference/plugins/create.md`. When porting, **split** a WHMCS module's `hooks.php` out into a
companion plugin; the provisioning logic stays in the module.

| WHMCS hook (examples) | Blesta home |
|---|---|
| `ClientEdit`, `AfterModuleCreate`, service lifecycle hooks | plugin `getEvents()` → `Clients.edit{Before,After}`, `Services.add{Before,After}`, etc. |
| `ClientAreaPrimaryNavbar` add child | plugin `getActions()` `nav_client` |
| `ClientAreaSecondarySidebar` panel | plugin card (`getCards()`) or client widget action |

---

## 10. Registrar modules (`RegistrarModule`)

A WHMCS registrar maps onto Blesta's `RegistrarModule` (`create.md` §15). The generic lifecycle still
applies (`addService` calls register/transfer; `renewService` calls renew), plus the domain surface:

| WHMCS registrar function | Blesta `RegistrarModule` | Notes |
|---|---|---|
| `_getConfigArray()` | **module-row meta** via `manageAddRow()`/`addModuleRow()` for API **credentials** (stored `encrypted => 1`); `getPackageFields()` only for per-package/TLD behaviour | WHMCS registrar config is module-level. Put API keys/credentials in a **module row**, not package meta — package fields are duplicated per package and aren't the place for secrets (`create.md` §0 layers, §7) |
| `_MetaData()` | `config.json` (`"type":"registrar"`, `features` block) | `features` gates `supports*` (`create.md` §15.1) |
| `_RegisterDomain($params)` | `registerDomain($domain, $module_row_id, $vars)` (called from `addService`) | |
| `_TransferDomain($params)` | `transferDomain($domain, $module_row_id, $vars)` | |
| `_RenewDomain($params)` | `renewDomain($domain, $module_row_id, $vars)` (from `renewService`) | |
| `_GetNameservers($params)` | `getDomainNameServers($domain, $module_row_id)` | |
| `_SaveNameservers($params)` | `setDomainNameservers($domain, $module_row_id, $vars)` | |
| `_GetContactDetails($params)` | `getDomainContacts($domain, $module_row_id)` | contact array shape in `create.md` §15.2 |
| `_SaveContactDetails($params)` | `setDomainContacts($domain, $vars, $module_row_id)` | |
| `_CheckAvailability($params)` | `checkAvailability($domain, $module_row_id)` (+ `checkTransferAvailability`) | |
| `_GetRegistrarLock($params)` | `getDomainIsLocked($domain, $module_row_id)` | |
| `_SaveRegistrarLock($params)` | `lockDomain()` / `unlockDomain()` | WHMCS one fn (lock value in params) → Blesta two methods |
| `_GetEPPCode($params)` | **No `getEppCode()` exists.** Blesta has `sendEppEmail($domain, $module_row_id)` (registry emails the code) and `updateEppCode($domain, $epp_code, …)` (set it); `supportsEppCode()` gates the feature | To *display* a code, fetch it via your API inside a service tab |
| `_IDProtectToggle($params)` | gated by `features.id_protection` / `supportsIdProtection()` + a toggle you implement (e.g. in a tab or `editService`) | No dedicated base toggle method — implement it |
| `_Sync($params)` / `_TransferSync($params)` | `getExpirationDate($service)` / `getRegistrationDate($service)` + `getDomainInfo($domain, $module_row_id)`, plus the **Domains plugin** sync (cron) | Blesta caches dates via the Domains plugin; override the date getters to query the registrar (`create.md` §15.4) |
| `_GetDNS` / `_SaveDNS` | **No `get/setDnsRecords()` on the base class** — only `supportsDnsManagement()` gates the feature; DNS records are managed by the **Domains plugin** | Implement DNS editing as a service tab if your registrar needs it |
| `_RegisterNameserver` / `_ModifyNameserver` / `_DeleteNameserver` | **No private-nameserver-host methods.** Closest is `setNameserverIps(array $vars, $module_row_id)` (glue/IP records) | Implement host register/modify/delete as a tab/action if needed |
| `_DomainSuggestionOptions` / `_GetDomainSuggestions` | **No core `RegistrarModule` method** — suggestions are handled by the order form / Domains plugin | Omit, or provide via the companion plugin |
| `_ReleaseDomain` / `_RequestDelete` | **No core equivalent.** (`restoreDomain()` exists for redemption, which is the inverse) | Map to a custom tab/action or omit |
| `_ClientArea` / `_ClientAreaCustomButtonArray` / `_ClientAreaAllowedFunctions` | `getClientServiceTabs()` + tab methods | §6 |

The **verified** `RegistrarModule` base surface (from `components/modules/registrar_module.php`) is:
`supportsDnsManagement/EmailForwarding/IdProtection/EppCode/Feature`, `checkAvailability`,
`checkTransferAvailability`, `isValidTerm`, `getDomainContacts`, `setDomainContacts`, `getDomainInfo`,
`getDomainIsLocked`, `getDomainNameServers`, `setDomainNameservers`, `setNameserverIps`,
`getRegistrationDate`, `getExpirationDate`, `lockDomain`, `unlockDomain`, `registerDomain`,
`transferDomain`, `resendTransferEmail`, `sendEppEmail`, `updateEppCode`, `renewDomain`, `restoreDomain`,
`getServiceDomain`, `getTlds`, `getTldPricing`, `getFilteredTldPricing`, `bulkCheckAvailability`.

> **Several WHMCS registrar functions have NO 1:1 Blesta method** — EPP *retrieval*, DNS *records*,
> private-nameserver *hosts*, domain *suggestions*, and *release/delete*. That's expected: Blesta pushes
> DNS, forwarding, and suggestions into the **Domains plugin** and the order form rather than the registrar
> module. For those, implement a service **tab/action** or omit the feature — **never call a base method
> that isn't in the verified list above** (a fatal call to a non-existent method is the classic porting bug).

The `getTldPricing()` shape (`[tld][currency][year]['register'|'transfer'|'renew']`) is Blesta-specific —
WHMCS has no equivalent function; you build it from your registrar's pricing API (`create.md` §15.2).

---

## 11. Conversion checklist

- [ ] Module rewritten as a **class** (`extends Module`, or `RegistrarModule` + `"type":"registrar"`),
      procedural `modulename_*` functions mapped to methods (§3 / §10).
- [ ] `whmcs.json`/`_MetaData()` → `config.json` (lang-key `name`/`description`, `icon`, `module` block) +
      `composer.json` `type: blesta-module`.
- [ ] **No `$params`.** Every value re-sourced: server creds from `$module_row->meta`, service data from
      `serviceFieldsToObject($service->fields)`, client from the `Clients` model, plan from `$package->meta`
      (§2).
- [ ] `_ConfigOptions()` positional `configoptionX` → **named** package-meta via `getPackageFields()`, with
      a documented index→name map (§4). Configurable add-ons → `$vars['configoptions']` by name.
- [ ] Secrets stored `encrypted => 1` (server password/key, service password).
- [ ] Errors via `$this->Input->setErrors()` (not a returned string); `logModuleCall` → paired
      `$this->log()` input/output with secrets masked (§3.1).
- [ ] `_CreateAccount` returns a **service-meta array** and honours `$vars['use_module']==='true'`;
      Terminate→`cancelService`; ChangePassword→tab/`editService`; ChangePackage→`changeServicePackage`.
- [ ] Server admin UI built (`manageModule`/`manageAddRow`/`manageEditRow`); `_TestConnection` folded into
      row validation.
- [ ] Order/edit service form built explicitly (`getAdmin/ClientAddFields`, `validateService`) — WHMCS
      generated this for you (§7).
- [ ] Custom buttons / client area / admin tab fields → Blesta **service tabs** + `getAdmin/ClientServiceInfo`;
      `.tpl` re-authored as `.pdt` (admin = BS5/Paradigm, client = BS4) (§6).
- [ ] `hooks.php` extracted into a **companion plugin** (events/actions/cards) — not the module (§9).
- [ ] Registrar: WHMCS domain functions mapped to verified `RegistrarModule` methods; unsupported ones
      implemented as tabs or omitted (no calls to non-existent base methods) (§10).

---

## 12. See also

- `create.md` — the full Blesta module contract (generic + registrar). **The mapping targets in this file
  all reference it.**
- `reference/plugins/create.md` — where a WHMCS module's `hooks.php` belongs.
- `reference/common/conversion-ui.md` — admin (Paradigm/BS5) view conversion, for any admin `.pdt` you author.
- WHMCS sample modules (`sample-provisioning-module`, `sample-registrar-module`) and
  `developers.whmcs.com/provisioning-modules/` / `/domain-registrars/` — the WHMCS-side source of truth.
