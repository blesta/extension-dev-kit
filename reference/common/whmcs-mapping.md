# Shared WHMCS → Blesta 6 Conversion Conventions

> **What this is:** the handful of conventions that are the **same across every WHMCS→Blesta port** —
> provisioning/registrar modules, gateways, and addons. The three `convert-from-whmcs.md` files
> (`reference/modules/`, `reference/gateways/`, `reference/plugins/`) each link here for these shared rules
> and keep only their **type-specific** mapping (the WHMCS `$params`/`$vars`, the function→method tables,
> return shapes, routing). Read this once; read the per-type doc for the actual port.

---

## 1. Licensing & sourcing caveat

The WHMCS SDK sample modules (`sample-provisioning-module`, `sample-registrar-module`,
`sample-gateway-module`, `sample-merchant-gateway`, `sample-addon-module`) ship an **MIT `LICENSE`** file,
**but the individual PHP files carry a conflicting WHMCS EULA header** (`@license … WHMCS Eula`). Given that
ambiguity:

- Use the samples and `developers.whmcs.com` only to learn the **contract** — function names, parameter
  shapes, field types, return shapes. Those are **facts**, not copyrightable expression.
- Write **original** Blesta-side code. **Do not copy WHMCS sample code verbatim** into a ported extension or
  into this kit.

---

## 2. The cardinal rule — re-source, don't transform

Every WHMCS function receives **one flat array** (`$params` for modules/gateways, `$vars` for addons)
carrying everything at once. Blesta instead hands you **structured objects and typed method arguments**.

> **There is no mechanical 1:1 transform of that flat array.** Re-source each value from the *right* Blesta
> place rather than reconstructing a single bag of data. The per-type doc has the exact table:
>
> - **Modules:** `$params` → `$package` / `$service` / `$module_row->meta` / `$vars` / the `Clients` model
>   (`reference/modules/convert-from-whmcs.md` §2).
> - **Gateways:** `$params` → method args + `$this->meta`
>   (`reference/gateways/convert-from-whmcs.md` §2).
> - **Addons:** `$vars` → your own settings storage + plugin context
>   (`reference/plugins/convert-from-whmcs.md` §2).

Likewise, WHMCS is **procedural** (global `modulename_Function()` dispatched by naming convention) and
Blesta is **object-oriented** (overridden methods on a class). You are **rewriting**, not editing.

---

## 3. Config field types → Blesta form helpers

WHMCS config blocks (`_ConfigOptions()` for provisioning, `_config()` for gateways/addons) expose **settings
fields** in the same six types. They map to Blesta's `InputFields`/`Form` helpers identically everywhere.
(Gateway `_config()` also carries a `FriendlyName` entry with `Type => 'System'` — that's the gateway's
display **name metadata**, not a settings field; it maps to `config.json` `name`, not a form input.)

| WHMCS `Type` | Blesta helper | Notes |
|---|---|---|
| `text` | `fieldText()` | |
| `password` | `fieldPassword()` | also mark the stored value as a **secret** (encrypt it) |
| `yesno` | `fieldCheckbox()` | value `'1'`/`'0'` |
| `dropdown` | `fieldSelect()` | assoc `Options` → option array |
| `radio` | `fieldRadio()` | CSV `Options` → option array |
| `textarea` | `fieldTextarea()` | `Rows`/`Cols` → attributes |

Common per-field attributes (`Size`, `Default`, `Description`, `Rows`, `Cols`) become field attributes /
default values / tooltips in Blesta.

> **Key by a stable machine name, not a label.** Two WHMCS traps:
>
> - **Addon/gateway `_config()` `fields`** is an associative array whose **key** is the config key;
>   `FriendlyName` is separate *display* metadata. The samples mislead by setting both to the same string
>   (`'Account ID' => ['FriendlyName' => 'Account ID', …]`). Use a stable machine key (`account_id`) in
>   Blesta.
> - **Provisioning `_ConfigOptions()`** is **positional** — code reads `$params['configoption1']`,
>   `configoption3`, so reordering silently breaks it. In Blesta give each a **name** (`$package->meta->disk`)
>   and record the index→name map during the port (`reference/modules/convert-from-whmcs.md` §4).
>
> Either way: the WHMCS **label** is not the contract; choose stable Blesta keys.

(WHMCS's other "options" concepts — *Configurable Options* and *Custom Fields* — are module-specific and
collapse differently onto Blesta; see the modules doc §4.)

---

## 4. Errors — return strings/status → `Input` errors

WHMCS signals failure by **returning** a value (the string `'success'`/an error message for provisioning, a
`['status' => 'success'|'declined'|'error']` array for gateways, a `['status' => …]` array for addon
activate/deactivate). Blesta signals failure by **setting Input errors**:

```php
$this->Input->setErrors(['field' => ['rule' => $message]]);
if ($this->Input->errors()) { return; }
```

The host app reads `->errors()` to decide whether to proceed. **Translate a returned error into a
`setErrors()` call and abort**, rather than returning a status string. (The one place an array is still
*returned* on failure is gateway processing of a genuine processor decline — see the gateways doc §7, which
also covers when *not* to return an array.)

---

## 5. Logging — `logModuleCall` / `logTransaction`

WHMCS logs remote calls with `logModuleCall(module, func, request, response, trace)` (modules) or
`logTransaction(name, data, status)` (gateways). The Blesta equivalent **depends on the extension type** —
`$this->log()` only exists on the module and gateway base classes.

**Modules & gateways** — use the inherited `protected log()` (verified in `components/modules/module.php`
and `components/gateways/lib/gateway.php`); it writes the admin-visible module/gateway log:

```php
$this->log($url, serialize($masked_request),  'input',  true);   // the request
$this->log($url, serialize($response),         'output', $success); // the response
```

- One WHMCS call → typically **two** Blesta log entries (`'input'` then `'output'`).
- `$url` is a label; the convention is `"$host|$operation"`.
- **Mask secrets first** — PAN/CVV, passwords, API keys (`maskData()`/`maskDataRecursive()` exist on the
  gateway base class). WHMCS's "raw response in the return" likewise becomes a logged value, not a returned
  field.

> **Plugins have NO `$this->log()`.** The `Plugin` base class (`components/plugins/lib/plugin.php`) does
> **not** define `log()` — don't call it from a converted addon. Instead use one of the patterns the shipped
> plugins use:
>
> - **App logger** for diagnostics: `$this->logger = $this->getFromContainer('logger');` then
>   `$this->logger->error(...)` / `->info(...)` (as `support_manager` does).
> - **A plugin-owned log table/model** when you need user-visible, queryable history: create the table in
>   `install()` and write through your own model (as `webhooks` does with its `WebhooksLogs` model).
>
> Pick the app logger for "something went wrong" diagnostics; pick a plugin-owned table for an audit trail
> the admin/client should see.

---

## 6. Templates — Smarty `.tpl` → Blesta `.pdt`, and the theme split

WHMCS client/admin output is **Smarty** (`.tpl`); Blesta uses its own PHP-based **`.pdt`** engine. This is a
**rewrite, not a syntax swap**:

| Smarty | Blesta `.pdt` |
|---|---|
| `{$var}` | `<?php echo $var; ?>` (escape with the `Html` helper as appropriate) |
| `{$LANG.x}` / language vars | `$this->_('Key')` / `Language::_('Key', true)` |
| `{if}…{/if}`, `{foreach}` | plain PHP `if`/`foreach` |

And the **admin/client theme split applies to every extension type** in 6.0:

- **Admin** views → **Paradigm / Bootstrap 5**, Bootstrap Icons (`bi bi-…`), **vanilla JS (no jQuery)**,
  must work in **light and dark mode**. Follow `reference/common/conversion-ui.md`.
- **Client** views → the `bootstrap` portal, still **Bootstrap 4 + jQuery + FontAwesome** in 6.0. Leave
  client views on BS4 (a BS5 client theme arrives in 6.1).

Ship the extension's own CSS/JS **inside the extension** and reference it directly from views — extension
assets do **not** go through the core Gulp build, and you never edit core assets.

---

## 7. Naming & packaging

- WHMCS module/file names are all-lowercase letters+numbers. Blesta class file names round-trip via
  `Loader::fromCamelCase()` (file `your_thing.php` ↔ class `YourThing`).
- `composer.json` `type` selects where Blesta installs the extension:
  `blesta-module`, `blesta-gateway-merchant`, `blesta-gateway-nonmerchant`, or `blesta-plugin`.
- WHMCS `whmcs.json` / `_config()`/`_MetaData()` metadata → Blesta `config.json` (with `name`/`description`
  as **language keys**). The per-type doc covers the exact `config.json` shape.

---

## 8. Where to go next

| Porting a… | Read |
|---|---|
| WHMCS provisioning/registrar module | `reference/modules/convert-from-whmcs.md` (+ `reference/modules/create.md`) |
| WHMCS gateway (third-party/merchant) | `reference/gateways/convert-from-whmcs.md` (+ `reference/gateways/create.md`) |
| WHMCS addon module | `reference/plugins/convert-from-whmcs.md` (+ `reference/plugins/create.md`) |
| Any admin view | `reference/common/conversion-ui.md` |
