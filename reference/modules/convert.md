# Porting a Blesta Module from 5.x to 6.0

> **Audience:** Developers (and AI tools) converting an existing 5.x module to run natively in Blesta 6.0.
>
> Conversion is **two independent jobs**: (1) the **UI/view layer** (Bootstrap 4→5, jQuery→vanilla,
> FontAwesome→Bootstrap Icons, Paradigm markup) and (2) the **PHP contract** (small, mostly backwards
> compatible). Do them in that order — the UI is 90% of the work.

---

## 1. Will the old module even run?

Yes — the `Module` / `RegistrarModule` contracts are **backwards compatible**. A 5.x module loads in 6.0:

- `getAdminTabs($package)` / `getClientTabs($package)` still work (the new
  `getAdminServiceTabs($service)` / `getClientServiceTabs($service)` fall through to them).
- `arrayToModuleFields()` still works (deprecated alias of `arrayToInputFields()`).
- `ModuleFields` still works alongside the newer `InputFields`.

What *looks* broken without conversion is the **admin** UI: a 5.x module renders with Bootstrap 4 markup,
FontAwesome icon classes, and jQuery handlers, all of which are wrong/absent in the Paradigm admin theme.
A SCSS backwards-compat shim keeps it *functional* but visually out of place.

> **Only the ADMIN side needs BS5 conversion.** The 6.0 **client portal** (`bootstrap` theme) is still
> **Bootstrap 4 + jQuery + FontAwesome**, so a module's client-facing views (`getClient*` methods,
> `tab_client_*.pdt`, `client_service_info.pdt`) should **stay BS4** — converting them to BS5 would break
> them. See `create.md` §12 for the full admin-vs-client table. (A BS5 client theme is planned for 6.1.)

> **Do NOT use the shipped core modules as a UI reference.** Contrary to an earlier draft of this guide,
> `cpanel` and `namesilo` are **not** fully converted in 6.0.0-b2: their PHP still emits FontAwesome icon
> strings (e.g. `getClientTabs` returns `'icon' => 'fas fa-cog'`, `cpanel.php`) and jQuery
> (`getPackageFields` uses `$(document).ready`, `cpanel.php`/`namesilo.php`), and their client views are
> BS4. For **admin** Paradigm patterns, reference the converted **plugins** (`order`, `support_manager`,
> `cms`) and `reference/common/conversion-ui.md` instead.

---

## 2. The UI conversion (the big job)

This is identical to converting a plugin's admin views. **Follow `reference/common/conversion-ui.md`
in full.** The high-frequency changes:

1. Bootstrap data attributes get the `-bs-` infix (`data-toggle` → `data-bs-toggle`, etc.).
2. Utility renames (`pull-right`→`float-end`, `ml-*`→`ms-*`, `text-right`→`text-end`, `font-weight-bold`→`fw-bold`, `sr-only`→`visually-hidden`).
3. Buttons (`btn-default`→`btn-secondary`/`btn-outline-secondary`, `btn-block`→`w-100`).
4. Badges (`badge-x`→`bg-x`; yellow needs `text-dark`).
5. Form controls — pass the BS5 class on every helper call (`form-control` / `form-select` /
   `form-check-input`), replace `<ul><li>` field rows with `<div class="mb-3">`, `form-group`→`mb-3`.
6. Cards — `inner`/`pad`/`title_row`/`button_row` → BS5 `card-body`/`form-section-header`/`card-footer`.
7. Tables — wrap in `table-responsive`, real `thead`/`tbody`, drop manual zebra striping.
8. Icons — every `fa*`/`fas fa-…` → `bi bi-…` (mapping table in the UI guide §5).
9. JavaScript — `$(document).ready` → `DOMContentLoaded` + `blestaAjaxComplete`; all jQuery → vanilla;
   `blestaRequest` → `fetch`; Bootstrap component access via `bootstrap.Modal/Tab/...`.
10. AJAX swap target `.common_box_content` → `.card-content`.
11. Test in **light and dark** mode (`<html data-bs-theme="dark">`).

**Convert ADMIN views only**: `manage.pdt`, `add_row.pdt`, `edit_row.pdt`, `admin_service_info.pdt`, and the
admin tab views/method output (e.g. `tab_stats.pdt`). **Do NOT convert client views** —
`client_service_info.pdt` and every `tab_client_*.pdt` stay Bootstrap 4 + jQuery + FontAwesome for the 6.0
client portal (see §1).

---

## 3. The PHP conversion (the small job)

### 3.1 Rename the field-builder override (optional)

If you override `arrayToModuleFields()`, rename it to `arrayToInputFields()`. The old name still works as a
deprecated alias, so this is cosmetic.

### 3.2 Replace jQuery emitted from PHP (admin-rendered builders only)

**Admin-rendered** field builders — `getPackageFields()`, `getAdminAddFields()`, `getAdminEditFields()` —
often build a `setHtml('<script>$(...)...</script>')` blob to show/hide fields. The **admin** theme
(Paradigm) has **no jQuery**, so rewrite these as vanilla JS. (`getClientAddFields()` and other
client-rendered builders run in the 6.0 client `bootstrap` theme, which **does** load jQuery — leave those
as-is for 6.0; see §1 and `create.md` §12.)

```diff
- $('#x').closest('li').hide();
- $('input[name="meta[type]"]').change(function(){ ... });
+ document.getElementById('x')?.closest('.mb-3')?.classList.add('d-none');
+ document.querySelectorAll('input[name="meta[type]"]').forEach(function(el){
+     el.addEventListener('change', function(){ ... });
+ });
```

Note the structural change too: fields are no longer wrapped in `<li>`, so `.closest('li')` becomes
`.closest('.mb-3')` (or whatever wrapper your converted markup uses).

### 3.3 Icon strings in PHP arrays

- **Admin-side icons** — `getAdminTabs` / `getAdminServiceTabs` array values and any `'icon' => 'fas fa-…'`
  in admin link-button/widget arrays → Bootstrap Icons `'icon' => 'bi bi-…'`.
- **Client-side icons** — `getClientTabs` / `getClientServiceTabs` array values render in the 6.0 client
  `bootstrap` theme, which still uses **FontAwesome** — so for 6.0, **leave client tab icons as `fas fa-…`**.
- **`config.json` `"icon"`** — add the **name only**, e.g. `"bi-server"` (the admin module list prepends the
  `bi` base class itself; `"bi bi-server"` here is wrong). See `create.md` §3.

### 3.4 Adopt the service-aware tab methods (optional, recommended)

Prefer `getAdminServiceTabs($service)` / `getClientServiceTabs($service)` over the package-aware
`getAdminTabs($package)` / `getClientTabs($package)`. The service-aware versions let you prune tabs based
on the specific service (e.g. registrar feature flags). The old methods keep working as a fallback.

### 3.5 Registrar modules

If your module is a registrar and predates the `RegistrarModule` base class, make sure:

- The class `extends RegistrarModule` (not `Module`).
- `config.json` has `"type": "registrar"` and an accurate `features` block — the host app reads `features`
  to decide which configurable options/tabs to show. Map each `1` to a method you actually implement.
- Domain methods match the current signatures in `create.md` §15.2 (e.g.
  `setDomainNameservers($domain, $module_row_id = null, array $vars = [])`).

---

## 4. What does *not* change

- Service/package/row **meta return shape** (`['key'=>,'value'=>,'encrypted'=>]`).
- The lifecycle method names and the `null`-keeps-meta / array-replaces-meta semantics.
- `$this->log(...)`, `$this->Input->setErrors(...)`, `getCommonError(...)`, `setMessage(...)`.
- Language file location/format and the `!error.` key convention.
- `apis/`, `config/`, `language/`, `views/default/` directory layout.

---

## 5. Conversion checklist

- [ ] **Admin** `.pdt` views converted per `reference/common/conversion-ui.md` (BS5, BI, vanilla JS).
- [ ] **Client** `.pdt` views left on BS4 + jQuery + FontAwesome (correct for the 6.0 client portal — do **not** convert).
- [ ] Inline `setHtml()` JS in **admin** field builders (`getPackageFields`/`getAdmin*Fields`) rewritten to vanilla (and `.closest('li')` → `.closest('.mb-3')`); client field-builder JS left as-is.
- [ ] Admin icons `'fas fa-…'` → `'bi bi-…'`; **client** tab icons left as `'fas fa-…'`; `"icon"` added to `config.json` as the **name only** (`bi-server`).
- [ ] `arrayToModuleFields` override renamed to `arrayToInputFields` (optional).
- [ ] Registrar: `extends RegistrarModule`, `"type":"registrar"`, accurate `features` block.
- [ ] **Admin** views verified in light **and** dark mode.
- [ ] No remaining jQuery in **admin** views/builders (`$`, `jQuery`, `.blestaXxx(` plugin calls).

---

## 6. See also

- `create.md` — the full method-surface reference (use it to confirm any signature/return shape).
- `reference/common/conversion-ui.md` — the exhaustive UI conversion guide (shared with plugins).
