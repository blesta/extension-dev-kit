# Blesta 6 Plugin & Module Conversion Guide

> **Audience:** Plugin and module developers porting an extension from Blesta 5.x to Blesta 6.
>
> **Purpose:** A single, exhaustive reference that an LLM (Claude, GPT, etc.) can load as context to convert an existing extension's admin views in one shot. Sections are independently consumable: tables, before/after snippets, and rules — no narrative.
>
> **Source:** Distilled from a side-by-side comparison of the Blesta 5.13 and Blesta 6 codebases for `support_manager`, `order`, `cms`, `feed_reader`, `system_overview`, and the core module base classes.
>
> **⚠ Scope — this guide is for the ADMIN area only.** Every rule here targets the **Paradigm** admin theme
> (Bootstrap 5 / Bootstrap Icons / vanilla JS). The Blesta **6.0 client portal** (`bootstrap` theme) is
> still **Bootstrap 4 + jQuery + FontAwesome** — do **not** apply these conversions to client-facing views
> (`getClient*` outputs, `tab_client_*.pdt`, `client_service_info.pdt`); they should stay BS4 for 6.0. A
> BS5 client theme is planned for 6.1. See `reference/modules/create.md` §12 for the admin-vs-client table.

---

## Table of Contents

1. [What changed at a glance](#1-what-changed-at-a-glance)
2. [Conversion checklist (TL;DR)](#2-conversion-checklist-tldr)
3. [Admin theme: `default` → `paradigm`](#3-admin-theme-default--paradigm)
4. [Bootstrap 4 → Bootstrap 5 class changes](#4-bootstrap-4--bootstrap-5-class-changes)
5. [Icons: FontAwesome → Bootstrap Icons](#5-icons-fontawesome--bootstrap-icons)
6. [JavaScript: jQuery → vanilla JS](#6-javascript-jquery--vanilla-js)
7. [HTML / markup structural changes](#7-html--markup-structural-changes)
8. [Form helper usage](#8-form-helper-usage)
9. [Widget helper changes](#9-widget-helper-changes)
10. [The new sidebar pattern (Paradigm)](#10-the-new-sidebar-pattern-paradigm)
11. [CSS, SASS, and theme variables](#11-css-sass-and-theme-variables)
12. [AJAX patterns](#12-ajax-patterns)
13. [Module-specific changes (PHP base classes)](#13-module-specific-changes-php-base-classes)
14. [Composer / PHP / dependency notes](#14-composer--php--dependency-notes)
15. [Common gotchas](#15-common-gotchas)
16. [Canonical reference files](#16-canonical-reference-files)
17. [LLM prompt template for one-shot conversion](#17-llm-prompt-template-for-one-shot-conversion)

---

## 1. What changed at a glance

| Layer | Blesta 5.x (`default` admin theme) | Blesta 6 (`paradigm` admin theme) |
|---|---|---|
| CSS framework | Bootstrap 4 | **Bootstrap 5** |
| Icon set | FontAwesome 5 (`fas fa-…`) | **Bootstrap Icons** (`bi bi-…`) |
| JavaScript | jQuery + custom plugins | **Vanilla JS** (no jQuery) |
| Theme switching | None | **Light / Dark mode** via `data-bs-theme` |
| Tooltips | `data-toggle="tooltip"` (auto-init) | `data-bs-toggle="tooltip"` (manual init) |
| Modal confirms | `$.fn.blestaModalConfirm` | `blestaModalConfirm()` global + declarative `.modal-confirm-delete` class |
| Markdown editor | CKEditor (`blestaBindWysiwygEditor`) | **Toast UI Editor** (`blestaBindMarkdownEditor`) |
| Sortable | jQuery UI (`blestaSortable`) | **Sortable.js** |
| AJAX widget swap target | `.common_box_content` | **`.card-content`** |
| Default PHP version | ≥ 7.2 | **≥ 8.2** |
| Forms inside cards | `<div class="inner">` + `<ul><li>` | `<div class="card-body">` + `<div class="mb-3">` |

**The legacy `default` admin theme still ships in Blesta 6** for backward compatibility, but `paradigm` is the active default. Plugins must render correctly in `paradigm`. A SCSS compatibility shim (`assets/sass/admin/paradigm/_backwards.scss`) keeps unconverted v5 plugins *functional* but they look out of place — convert them for a native experience.

**Modules in v6 core ship UNCONVERTED** (still BS4 + FA + jQuery). Don't model your conversion on the shipped core modules — model it on `app/views/admin/paradigm/admin_clients.pdt` or any of the converted plugins (`order`, `support_manager`, `cms`, `feed_reader`, `system_overview`).

---

## 2. Conversion checklist (TL;DR)

Apply these in order:

1. **Bootstrap data attributes**: every `data-toggle`, `data-target`, `data-dismiss`, `data-parent`, `data-placement`, `data-html`, `data-container`, `data-auto-close` gets a `-bs-` infix → `data-bs-toggle`, `data-bs-target`, etc.
2. **Float/margin/text utilities**: `pull-right`/`float-right` → `float-end`; `pull-left`/`float-left` → `float-start`; `ml-N`/`mr-N` → `ms-N`/`me-N`; `pl-N`/`pr-N` → `ps-N`/`pe-N`; `text-left`/`text-right` → `text-start`/`text-end`; `font-weight-bold` → `fw-bold`; `sr-only` → `visually-hidden`.
3. **Buttons**: `btn-default` → `btn-secondary` (or `btn-outline-secondary` for icon-only row actions); `btn-block` → `w-100` (or wrap in `d-grid`).
4. **Badges**: `badge badge-{color}` → `badge bg-{color}`. Yellow needs dark text: `badge bg-warning text-dark`.
5. **Form controls**: pass `'class' => 'form-control'` to text/textarea/file, `'class' => 'form-select'` to selects, `'class' => 'form-check-input'` to checkboxes/radios. Replace `form-group` with `mb-3`. Replace `<ul><li>` form rows with `<div class="mb-3">` blocks. Replace `custom-control` family with `form-check` family.
6. **Form labels**: pass `'class' => 'form-label'` to top-block labels, `'class' => 'form-check-label'` to checkbox/radio labels. Replace `<label class="inline">` with `<label class="form-check-label">`.
7. **Cards**: replace `<div class="inner">` / `<div class="pad">` / `<div class="title_row">` / `<div class="button_row">` with BS5 card sections (see §7.2).
8. **Tables**: wrap in `<div class="table-responsive">`; use `<table class="table table-hover mb-0">`; add proper `<thead>`/`<tbody>`; drop `class="heading_row"`, `class="last"`, manual `odd_row` zebra striping. In widget **listing views**, also call `$this->Widget->setBodyWrapper(false)` before `Widget->create()` so the table runs edge-to-edge (the default card-body wrapper otherwise insets it with padding), and give the empty-state branch its own `<div class="card-body text-center text-muted py-5">`.
9. **Modal close**: `<button class="close" data-dismiss="modal"><span>&times;</span></button>` → `<button class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>`.
10. **Icons**: replace every `<i class="fas|far|fa fa-…">` with `<i class="bi bi-…">` (see §5 for the mapping table).
11. **JavaScript**: replace every `$(document).ready(...)`, jQuery selector, jQuery event, `$.fn.blestaXxx`, `blestaRequest({dataType:'json'})` with vanilla JS equivalents (see §6).
12. **Widget icon registrations**: in `setLinkButtons`, `setLinks`, `setWidgetButton`, and any `'icon' => 'fas fa-…'` keys in PHP, change values to `bi bi-…`.
13. **Widget renamed buttons**: `setWidgetButton('full_screen')` → `setWidgetButton('full-screen')` (hyphen, not underscore). Optionally add `setWidgetButton('widget-width')` for dashboard widgets.
14. **AJAX swap target**: if your code emits an AJAX widget that replaces `.common_box_content`, change the target to `.card-content`.
15. **Module config.json**: optionally add `"icon": "bi-…"` (name only — Paradigm prepends the `bi` base class itself; do **not** write `"bi bi-…"` here) so the admin module list renders an icon. Inline `<i>` tags and tab/link `'icon'` arrays still use the full `bi bi-…`.
16. **Module PHP base class**: rename `arrayToModuleFields()` overrides to `arrayToInputFields()` (old name still works as a deprecated alias).
17. **Test in both light and dark mode** by toggling `<html data-bs-theme="dark">`/`"light"` in DevTools.

---

## 3. Admin theme: `default` → `paradigm`

### 3.1 Theme location

| | v5 (`default`) | v6 (`paradigm`) |
|---|---|---|
| Views | `public_html/app/views/admin/default/` | `public_html/app/views/admin/paradigm/` |
| SCSS | `assets/sass/admin/default/` | `assets/sass/admin/paradigm/` |
| JS | `assets/scripts/admin/default/` | `assets/scripts/admin/paradigm/` |
| Compiled CSS | `…/css/application.min.css` (gitignored, generated) | `…/css/application.min.css` (gitignored, generated) |
| Compiled JS | `…/javascript/app.min.js` | `…/javascript/app.min.js` |
| Theme manifest | n/a | `…/paradigm/config.json` declares assets to load |

The active layout is set in `public_html/app/app_controller.php`: `paradigm` for admin (with a `template_dir` company-level override). Do not assume `default` is the active admin theme.

### 3.2 Plugin `structure.pdt`

Plugin `views/default/structure.pdt` is unchanged — it remains a pass-through `<?php echo $content ?? null;?>`. Plugins delegate chrome to the host theme. **All your conversion work is in your action `.pdt` files.**

### 3.3 New partials directory

Paradigm extracts shared chrome into `app/views/admin/paradigm/partials/`:

- `sidebar.pdt`, `notifications.pdt`, `admin_query_logger.pdt`
- `main_sidebar.pdt`, `admin_clients_sidebar.pdt`, `admin_settings_sidebar.pdt`, `admin_chatbot_sidebar.pdt`, **`plugin_sidebar.pdt`**

`plugin_sidebar.pdt` is the slot your plugin renders into when registering a side panel — see §10.

---

## 4. Bootstrap 4 → Bootstrap 5 class changes

### 4.1 Float / spacing / text utilities

| BS4 | BS5 |
|---|---|
| `pull-right`, `float-right` | `float-end` |
| `pull-left`, `float-left` | `float-start` |
| `ml-N` (1–5) | `ms-N` |
| `mr-N` | `me-N` |
| `pl-N` | `ps-N` |
| `pr-N` | `pe-N` |
| `text-left` | `text-start` |
| `text-right` | `text-end` |
| `font-weight-bold` | `fw-bold` |
| `font-weight-normal` | `fw-normal` |
| `font-weight-light` | `fw-light` |
| `sr-only` | `visually-hidden` |
| `sr-only-focusable` | `visually-hidden-focusable` |
| `text-monospace` | `font-monospace` |

`mt-N`, `mb-N`, `pt-N`, `pb-N`, `m-N`, `p-N` are unchanged.

### 4.2 Buttons

| BS4 | BS5 |
|---|---|
| `btn-default` | `btn-secondary` (or `btn-outline-secondary` for icon-only) |
| `btn-block` | `w-100` (or wrap parent in `d-grid`) |

```diff
- <a class="btn btn-default float-right btn-sm" href="#">Cancel</a>
+ <a class="btn btn-outline-secondary float-end btn-sm" href="#">Cancel</a>

- <button class="btn btn-block btn-info">Action</button>
+ <button class="btn w-100 btn-info">Action</button>
```

Row action buttons are now icon-only outline buttons:

```html
<a href="<?php echo $edit_url;?>" class="btn btn-sm btn-outline-secondary" title="Edit">
    <i class="bi bi-pencil"></i>
</a>
<button type="button" class="btn btn-sm btn-outline-danger modal-confirm-delete"
        data-confirm-message="<?php echo $this->Html->safe($confirm_msg);?>"
        title="Delete">
    <i class="bi bi-trash"></i>
</button>
```

### 4.3 Badges

```diff
- <span class="badge badge-secondary">5</span>
- <span class="badge badge-info">Info</span>
- <span class="badge badge-warning">Warn</span>
+ <span class="badge bg-secondary">5</span>
+ <span class="badge bg-info text-dark">Info</span>
+ <span class="badge bg-warning text-dark">Warn</span>
```

The yellow palette (`bg-warning`, `bg-info`, `bg-light`) needs `text-dark` for contrast in BS5.

Tab counters become real badges:

```diff
- 'name' => $label . ' <span>(' . $count . ')</span>'
+ 'name' => $label . ' <span class="badge bg-secondary ms-2">' . $count . '</span>'
```

A typical priority/status mapping helper:

```php
$priority_classes = [
    'emergency' => 'bg-danger',
    'critical'  => 'bg-warning text-dark',
    'high'      => 'bg-info text-dark',
    'medium'    => 'bg-primary',
    'low'       => 'bg-secondary'
];
$badge_class = $priority_classes[$ticket->priority] ?? 'bg-secondary';
?>
<span class="badge <?php echo $badge_class;?>"><?php echo $priorities[$ticket->priority];?></span>
```

### 4.4 Bootstrap data attributes (the `-bs-` infix)

Every Bootstrap-controlled `data-*` attribute gets a `bs-` prefix:

| BS4 | BS5 |
|---|---|
| `data-toggle="tab"` | `data-bs-toggle="tab"` |
| `data-toggle="collapse"` | `data-bs-toggle="collapse"` |
| `data-toggle="tooltip"` | `data-bs-toggle="tooltip"` |
| `data-toggle="modal"` | `data-bs-toggle="modal"` |
| `data-toggle="dropdown"` | `data-bs-toggle="dropdown"` |
| `data-target="#x"` | `data-bs-target="#x"` |
| `data-dismiss="modal"` | `data-bs-dismiss="modal"` |
| `data-dismiss="alert"` | `data-bs-dismiss="alert"` |
| `data-parent="#x"` | `data-bs-parent="#x"` |
| `data-placement="top"` | `data-bs-placement="top"` |
| `data-html="true"` | `data-bs-html="true"` |
| `data-container="body"` | `data-bs-container="body"` |
| `data-auto-close="outside"` | `data-bs-auto-close="outside"` |

### 4.5 Forms — controls, groups, checks

| BS4 | BS5 |
|---|---|
| `<div class="form-group">` | `<div class="mb-3">` |
| `<div class="form-row">` | `<div class="row g-3">` |
| `<input class="form-control">` | unchanged |
| `<select class="form-control">` | `<select class="form-select">` |
| `<div class="custom-control custom-checkbox">` | `<div class="form-check">` |
| `<input class="custom-control-input">` | `<input class="form-check-input">` |
| `<label class="custom-control-label">` | `<label class="form-check-label">` |
| `<select class="custom-select">` | `<select class="form-select">` |
| `<input type="file" class="custom-file">` | `<input type="file" class="form-control">` |

The Blesta `Form` helper does **not** auto-inject BS5 classes. You must pass them on every call:

```diff
- $this->Form->fieldText('name', $val, ['id' => 'name']);
- $this->Form->fieldSelect('priority', $opts, $val, ['id' => 'priority']);
- $this->Form->fieldCheckbox('clients_only', '1', $checked, ['id' => 'clients_only']);
- $this->Form->label('Name', 'name');
+ $this->Form->fieldText('name', $val, ['id' => 'name', 'class' => 'form-control']);
+ $this->Form->fieldSelect('priority', $opts, $val, ['id' => 'priority', 'class' => 'form-select']);
+ $this->Form->fieldCheckbox('clients_only', '1', $checked, ['id' => 'clients_only', 'class' => 'form-check-input']);
+ $this->Form->label('Name', 'name', ['class' => 'form-label']);
```

### 4.6 Form layout: `<ul><li>` → `<div class="mb-3">`

The single largest pattern change. The legacy "list of fields" wrapper is gone:

```diff
- <div class="pad">
-     <ul>
-         <li>
-             <?php $this->Form->label('Name', 'name'); ?>
-             <?php $this->Form->fieldText('name', null, ['id' => 'name']); ?>
-         </li>
-         <li>
-             <?php $this->Form->fieldCheckbox('enabled', '1', $checked, ['id' => 'enabled']); ?>
-             <?php $this->Form->label('Enabled', 'enabled', ['class' => 'inline']); ?>
-         </li>
-     </ul>
- </div>
+ <div class="mb-3">
+     <?php $this->Form->label('Name', 'name', ['class' => 'form-label']); ?>
+     <?php $this->Form->fieldText('name', null, ['id' => 'name', 'class' => 'form-control']); ?>
+ </div>
+ <div class="mb-3">
+     <div class="form-check">
+         <?php $this->Form->fieldCheckbox('enabled', '1', $checked, ['id' => 'enabled', 'class' => 'form-check-input']); ?>
+         <?php $this->Form->label('Enabled', 'enabled', ['class' => 'form-check-label']); ?>
+     </div>
+ </div>
```

Inline checkbox/radio groups use `form-check-inline`:

```html
<div class="form-check form-check-inline">
    <?php $this->Form->fieldCheckbox('opt[a]', '1', $checked, [
        'id' => 'opt_a', 'class' => 'form-check-input'
    ]); ?>
    <label class="form-check-label" for="opt_a">Option A</label>
</div>
```

Toggle switches:

```html
<div class="form-check form-switch">
    <?php $this->Form->fieldCheckbox('feature_enabled', 'true', $vars->feature_enabled === 'true', [
        'id' => 'feature_enabled', 'class' => 'form-check-input'
    ]); ?>
    <label class="form-check-label" for="feature_enabled">Enable feature</label>
</div>
```

Bracketed names need underscore IDs (because `<label for="">` requires a valid HTML id):

```diff
- 'id' => 'settings[email][' . $key . ']'
- $this->Form->label($label, 'settings[email][' . $key . ']', ['class' => 'inline']);
+ 'id' => 'settings_email_' . $key
+ <label class="form-check-label" for="settings_email_<?php echo $key;?>"><?php echo $label;?></label>
```

### 4.7 Sized form controls

```html
<input class="form-control form-control-sm">
<select class="form-select form-select-sm">
<button class="btn btn-sm btn-primary">
```

### 4.8 Modal close button

```diff
- <button type="button" class="close" data-dismiss="modal"><span aria-hidden="true">&times;</span></button>
+ <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
```

`.close` is removed in BS5. The new `.btn-close` is an icon-only square button — no text content.

### 4.9 Tabs

Native BS5 tabs. The legacy Blesta `tab_content` / `tabs` / `current` markup (driven by jQuery `blestaTabbedContent`) is replaced:

```diff
- <div class="tab_content">
-     <ul class="tabs">
-         <li class="current"><a href="#">Reply</a></li>
-         <li><a href="#">Note</a></li>
-     </ul>
-     <div class="inner_content">
-         <div>...</div>
-         <div>...</div>
-     </div>
- </div>
+ <ul class="nav nav-tabs-connected" role="tablist">
+     <li class="nav-item" role="presentation">
+         <button class="nav-link active" id="tab-reply-btn" type="button" role="tab"
+                 data-bs-toggle="tab" data-bs-target="#tab-reply">Reply</button>
+     </li>
+     <li class="nav-item" role="presentation">
+         <button class="nav-link" id="tab-note-btn" type="button" role="tab"
+                 data-bs-toggle="tab" data-bs-target="#tab-note">Note</button>
+     </li>
+ </ul>
+ <div class="tab-content tab-content-connected mb-3">
+     <div class="tab-pane fade show active" id="tab-reply" role="tabpanel">...</div>
+     <div class="tab-pane fade" id="tab-note" role="tabpanel">...</div>
+ </div>
```

`nav-tabs-connected` and `tab-content-connected` are **Paradigm-specific** classes that flush-mount tabs against a card edge. Stock BS5 `nav-tabs` works too.

### 4.10 Tooltips

```diff
- <a data-toggle="tooltip" title="Help text">
-     <i class="fas fa-question-circle"></i>
- </a>
+ <i class="bi bi-info-circle ms-1"
+    data-bs-toggle="tooltip" data-bs-placement="top"
+    title="<?php echo $this->Html->safe($help_text);?>"></i>
```

Tooltips are no longer auto-initialized. After page load (and after AJAX swaps), explicitly init:

```js
document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(function(el) {
    new bootstrap.Tooltip(el);
});
```

For HTML content add `data-bs-html="true"`. For tooltips that escape `overflow:hidden` parents add `data-bs-container="body"`.

### 4.11 Backgrounds / theme-aware colors

```diff
- <div class="bg-light p-3">
+ <div class="bg-body-secondary p-3">

- <pre class="bg-light"><code>...</code></pre>
+ <pre class="bg-body-secondary"><code>...</code></pre>
```

`bg-body-secondary` is BS 5.3+ and is theme-aware (works in light and dark mode). `bg-light` is hard-coded light gray and looks broken in dark mode.

### 4.12 Column / grid

| BS4 | BS5 |
|---|---|
| `col-xs-N` | `col-N` (bare `col-` is now the smallest breakpoint) |
| `row.equal` (custom equal-height) | `row` + `g-N` gutter / Bootstrap's flex utilities |

```diff
- <div class="col-md-2 col-xs-4">
- <div class="row equal">
+ <div class="col-md-2 col-4">
+ <div class="row g-2">
```

### 4.13 Spinners

```html
<div class="spinner-border spinner-border-sm text-secondary" role="status">
    <span class="visually-hidden">Loading...</span>
</div>
```

### 4.14 Empty states

```diff
- <div class="empty_section">
-     <div class="empty_box">No results.</div>
- </div>
+ <div class="card-body text-center text-muted py-5">
+     <i class="bi bi-inbox" style="font-size: 3rem; opacity: 0.3;"></i>
+     <p class="mt-3 mb-0">No results.</p>
+ </div>
```

Pick a contextual icon: `bi-inbox` (tickets, lists), `bi-cart` (orders), `bi-cash-stack` (invoices/payouts), `bi-people` (clients/staff), `bi-box` (products), `bi-search` (search), `bi-file-earmark-text` (articles), `bi-building` (departments), `bi-input-cursor-text` (custom fields), `bi-calendar-week` (schedules).

### 4.15 Pagination

```diff
- <?php $this->Pagination->build();?>
+ <?php if (isset($this->Pagination) && $this->Pagination->hasPages()) { ?>
+ <div class="card-footer">
+     <?php $this->Pagination->build();?>
+ </div>
+ <?php } ?>
```

Wrap in a card-footer and guard with `hasPages()` so empty paginators don't render.

### 4.16 Bulk actions bar

```html
<div class="bulk-actions-bar" id="ticket_actions" style="display:none;">
    <div class="d-flex justify-content-between align-items-center flex-wrap gap-2">
        <span class="selected-count">0 items selected</span>
        <div class="d-flex gap-2 align-items-center flex-wrap">
            <?php $this->Form->fieldSelect('action', $actions, '', [
                'id' => 'ticket_action',
                'class' => 'form-select form-select-sm',
                'style' => 'width: auto;'
            ]); ?>
            <?php $this->Form->fieldSubmit('save', 'Update', ['class' => 'btn btn-sm btn-primary']); ?>
        </div>
    </div>
</div>
```

`.bulk-actions-bar` and `.selected-count` are Paradigm helper classes.

### 4.17 Misc

| Old | New |
|---|---|
| `<div class="clear"></div>` | (drop — flex/grid replaces clearfix needs) |
| `<div class="accent_box">` | `<div class="alert alert-info">` |
| `<div class="title_row [first]"><h3>…</h3></div>` | `<h6 class="form-section-header">…</h6>` |
| `<div class="thumbnail">` | `<div class="template-preview-card">` (Paradigm class) |
| `class="quiet"` | drop |
| `class="block"`, `class="inline"`, `class="stretch"` | drop or replace with utility (`d-block`, `d-inline`, `flex-grow-1`) |

---

## 5. Icons: FontAwesome → Bootstrap Icons

Every `<i class="fas|far|fa fa-…">` must become `<i class="bi bi-…">`. Drop `fa-fw` (no equivalent).

### 5.1 Comprehensive mapping

| FontAwesome 5/6 | Bootstrap Icons | Notes |
|---|---|---|
| `fas fa-plus` / `fa-plus fa-fw` | `bi bi-plus-lg` | "Add" buttons |
| `fas fa-plus-circle` | `bi bi-plus-circle` | Add row/option |
| `fas fa-minus` | `bi bi-dash-lg` | |
| `fas fa-times` | `bi bi-x-lg` | Close |
| `fas fa-times-circle` | `bi bi-x-circle` | Clear filter |
| `fas fa-check` | `bi bi-check-lg` | |
| `fas fa-check-circle` | `bi bi-check-circle` (or `bi-check-circle-fill text-success`) | |
| `fas fa-pencil-alt` / `fa-pencil` | `bi bi-pencil` | Edit |
| `fas fa-trash` / `fa-trash-alt` | `bi bi-trash` | Delete |
| `fas fa-cog` / `fa-gear` / `fa-cogs` | `bi bi-gear` (or `bi-gear-wide-connected`) | Settings |
| `fas fa-eye` | `bi bi-eye` | Visible |
| `fas fa-eye-slash` | `bi bi-eye-slash` | Hidden |
| `fas fa-info-circle` | `bi bi-info-circle` | Info / tooltip |
| `fas fa-exclamation-triangle` | `bi bi-exclamation-triangle` | Warning |
| `fas fa-question-circle` | `bi bi-question-circle` | Help tooltip |
| `fas fa-shield` / `fa-shield-alt` | `bi bi-shield-exclamation` | Security/CSRF warning |
| `fas fa-ban` | `bi bi-ban` (or `bi-x-circle`) | |
| `fas fa-arrow-up` | `bi bi-arrow-up` | |
| `fas fa-arrow-down` | `bi bi-arrow-down` | |
| `fas fa-arrow-left` | `bi bi-arrow-left` | Back |
| `fas fa-arrow-right` | `bi bi-arrow-right` | Forward |
| `fas fa-arrow-circle-left` | `bi bi-arrow-left-circle` | |
| `fas fa-arrow-circle-right` | `bi bi-arrow-right-circle` | |
| `fas fa-angle-left` / `fa-chevron-left` | `bi bi-chevron-left` | |
| `fas fa-angle-right` / `fa-chevron-right` | `bi bi-chevron-right` | |
| `fas fa-chevron-up` | `bi bi-chevron-up` | |
| `fas fa-chevron-down` | `bi bi-chevron-down` | |
| `fas fa-caret-down` | `bi bi-chevron-down` | Collapse caret |
| `fas fa-caret-up` | `bi bi-chevron-up` | |
| `fas fa-arrows-alt` | `bi bi-grip-vertical` | Drag handle (use `bi-grip-vertical` for sortable) |
| `fas fa-grip-vertical` | `bi bi-grip-vertical` | |
| `fas fa-arrows-fullscreen` | `bi bi-arrows-fullscreen` | |
| `fas fa-arrows-angle-contract` | `bi bi-arrows-angle-contract` | |
| `fas fa-arrows-angle-expand` | `bi bi-arrows-angle-expand` | |
| `fas fa-fullscreen-exit` | `bi bi-fullscreen-exit` | |
| `fas fa-level-up-alt fa-rotate-90` | `bi bi-arrow-90deg-up` | "Service info" header arrow |
| `fas fa-level-down-alt fa-rotate-180` | `bi bi-arrow-return-left` | |
| `fas fa-share fa-flip-vertical` | `bi bi-reply-fill` (or `bi-arrow-return-left`) | "Service info" client arrow |
| `fas fa-reply` | `bi bi-reply` | |
| `fas fa-redo-alt` / `fa-sync-alt` | `bi bi-arrow-clockwise` | Refresh / regenerate |
| `fas fa-exchange-alt` | `bi bi-arrow-left-right` | |
| `fas fa-link` | `bi bi-link-45deg` | |
| `fas fa-unlink` | `bi bi-link-45deg` (no exact unlink — combine with overlay) | |
| `fas fa-external-link-alt` | `bi bi-box-arrow-up-right` | External link |
| `fas fa-sign-in-alt` | `bi bi-box-arrow-in-right` | Login as client |
| `fas fa-sign-out-alt` | `bi bi-box-arrow-right` | |
| `fas fa-power-off` | `bi bi-power` | |
| `fas fa-play` | `bi bi-play-fill` | |
| `fas fa-stop` | `bi bi-stop-fill` | |
| `fas fa-lock` | `bi bi-lock` | |
| `fas fa-key` | `bi bi-key` | |
| `fas fa-envelope` / `fa-envelope-open-text` | `bi bi-envelope` | |
| `fas fa-calendar` | `bi bi-calendar3` | |
| `fas fa-calendar-x` | `bi bi-calendar-x` | |
| `fas fa-calendar-week` | `bi bi-calendar-week` | |
| `fas fa-clock` | `bi bi-clock` | |
| `fas fa-history` / `fa-clock-history` | `bi bi-clock-history` | |
| `fas fa-user` | `bi bi-person` | |
| `fas fa-users` | `bi bi-people` | |
| `fas fa-user-circle` | `bi bi-person-circle` | |
| `fas fa-tag` | `bi bi-tag` | |
| `fas fa-tags` | `bi bi-tags` | |
| `fas fa-folder` | `bi bi-folder2` (or `bi-folder-fill` for solid) | |
| `fas fa-folder-open` | `bi bi-folder2-open` | |
| `fas fa-folder-plus` | `bi bi-folder-plus` | |
| `fas fa-file` / `fa-file-alt` | `bi bi-file-text` (or `bi-file-earmark-text`) | |
| `fas fa-paperclip` | `bi bi-paperclip` | Attachments |
| `fas fa-image` | `bi bi-image` | |
| `fas fa-cloud-upload-alt` | `bi bi-cloud-upload` | File drop zone |
| `fas fa-upload` | `bi bi-upload` | |
| `fas fa-download` | `bi bi-download` | |
| `fas fa-clone` / `fa-copy` | `bi bi-files` (or `bi-clipboard`) | |
| `fas fa-clipboard` | `bi bi-clipboard` | |
| `fas fa-clipboard-check` | `bi bi-clipboard-check` | |
| `fas fa-search` | `bi bi-search` | |
| `fas fa-filter` | `bi bi-funnel` (or `bi-funnel-fill`) | |
| `fas fa-bars` | `bi bi-list` | |
| `fas fa-list` | `bi bi-list-ul` | |
| `fas fa-money-bill-wave` | `bi bi-cash-stack` | |
| `fas fa-dollar-sign` | `bi bi-currency-dollar` | |
| `fas fa-chart-bar` | `bi bi-bar-chart` | Stats |
| `fas fa-chart-line` | `bi bi-graph-up` | |
| `fas fa-chart-pie` | `bi bi-pie-chart` | |
| `fas fa-database` | `bi bi-database` | |
| `fas fa-server` | `bi bi-hdd-rack` | |
| `fas fa-hdd` | `bi bi-hdd` | |
| `fas fa-microchip` | `bi bi-cpu` | |
| `fas fa-ethernet` | `bi bi-ethernet` (or `bi-hdd-network`) | |
| `fas fa-globe` / `fa-globe-americas` | `bi bi-globe-americas` | |
| `fas fa-sitemap` | `bi bi-diagram-3` | |
| `fas fa-terminal` | `bi bi-terminal` | |
| `fas fa-music` | `bi bi-music-note` | |
| `fas fa-moon` | `bi bi-moon` | Dark mode toggle |
| `fas fa-sun` | `bi bi-sun` | Light mode toggle |
| `fas fa-bell` | `bi bi-bell` | |
| `fas fa-star` (filled) | `bi bi-star-fill` | |
| `far fa-star` (empty) | `bi bi-star` | |
| `fas fa-stars` | `bi bi-stars` | AI feature indicator |
| `fas fa-stickynote` / `fa-sticky-note` | `bi bi-sticky` | Internal notes |
| `fas fa-chat-square-text` | `bi bi-chat-square-text` | Predefined responses |
| `fas fa-chat-left-quote` | `bi bi-chat-left-quote` | Customer comment |
| `fas fa-comment` | `bi bi-chat` | |
| `fas fa-lightbulb` | `bi bi-lightbulb` | |
| `fas fa-certificate` | `bi bi-patch-check` (or `bi-award`) | |
| `fas fa-hand-point-right` | `bi bi-hand-index` | |
| `fas fa-shield-exclamation` | `bi bi-shield-exclamation` | |
| `fas fa-inbox` | `bi bi-inbox` | |
| `fas fa-building` | `bi bi-building` | |

### 5.2 Icon classes in PHP (ADMIN only)

The framework's **admin** `setLinkButtons`, `setLinks`, `setWidgetButton`, and **plugin**
`getAdminServiceTabs` arrays carry an `'icon'` key — convert these to Bootstrap Icons:

```diff
  $link_buttons[] = [
      'name' => $this->_('AdminMain.btn_add', true),
      'attributes' => ['href' => $url],
-     'icon' => 'fas fa-plus'
+     'icon' => 'bi bi-plus-lg'
  ];

  // PLUGIN php file — admin service tabs support the name/icon array form
- 'tabStats'    => ['name' => Language::_('MyPlugin.tab_stats', true),    'icon' => 'fas fa-cog'],
+ 'tabStats'    => ['name' => Language::_('MyPlugin.tab_stats', true),    'icon' => 'bi bi-bar-chart'],
```

> **⚠ MODULE admin tabs are different — plain strings ONLY.** A module's `getAdminTabs()` /
> `getAdminServiceTabs()` must return `'tabFoo' => "Title"`. Core (`AdminClients::getTabs`) uses the
> value directly as the tab label — the `['name' => …, 'icon' => …]` array form renders the tab as the
> literal text "Array" and logs `E_WARNING: Array to string conversion` in `helpers/widget/widget.php`.
> Module admin tabs cannot carry icons; every shipped v6 module (cpanel, pterodactyl, …) uses strings.

> **Do NOT convert client tab icons.** `getClientTabs` / `getClientServiceTabs` arrays render in the 6.0
> client portal (`bootstrap` theme), which still uses **FontAwesome** — leave those `'icon' => 'fas fa-…'`
> values as-is for 6.0.

### 5.3 Status icons with semantic colors

```diff
- <i class="fas fa-check-circle"></i>  <!-- CSS-styled green -->
- <i class="fas fa-times-circle"></i>  <!-- CSS-styled red -->
+ <i class="bi bi-check-circle text-success"></i>
+ <i class="bi bi-x-circle text-danger"></i>
```

For services:
- Active service: `bi bi-check-circle-fill text-success`
- Scheduled cancellation: `bi bi-calendar-x text-warning`
- Pending change (client): `bi bi-clock-history text-warning`
- Pending change (billing): `bi bi-clock text-info`

### 5.4 Empty state icons

`<i class="bi bi-inbox" style="font-size: 3rem; opacity: 0.3;"></i>` or `class="bi bi-inbox fs-1 mb-3 d-block opacity-50"`. Pick by context (see §4.14).

---

## 6. JavaScript: jQuery → vanilla JS

Paradigm bundles **no jQuery**. Every script in your converted views must be vanilla JS.

### 6.1 DOM ready

```diff
- $(document).ready(function() { ... });
+ document.addEventListener('DOMContentLoaded', function() { ... });
```

For widget/AJAX-loaded scripts that may run after DOMContentLoaded has already fired, use the IIFE pattern:

```js
(function() {
    'use strict';
    function init() { /* ... */ }
    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', init);
    } else {
        init();
    }
    document.addEventListener('blestaAjaxComplete', init);   // re-bind after AJAX swaps
})();
```

### 6.2 Selector / event idiom replacements

| jQuery | Vanilla JS |
|---|---|
| `$('#x')` | `document.getElementById('x')` |
| `$('.x')` | `document.querySelectorAll('.x')` |
| `$(el).find('.x')` | `el.querySelectorAll('.x')` |
| `$(el).closest('.x')` | `el.closest('.x')` |
| `$(el).val()` / `.val(v)` | `el.value` / `el.value = v` |
| `$(el).text(s)` | `el.textContent = s` |
| `$(el).html(s)` | `el.innerHTML = s` |
| `$(el).attr('href', x)` | `el.setAttribute('href', x)` (or `el.href = x`) |
| `$(el).prop('checked', true)` | `el.checked = true` |
| `$(el).prop('disabled', true)` | `el.disabled = true` |
| `$(el).addClass('x')` / `.removeClass('x')` / `.toggleClass('x')` | `el.classList.add/remove/toggle('x')` |
| `$(el).show()` / `.hide()` | `el.style.display = 'block'` / `'none'` (or `el.classList.toggle('d-none')`) |
| `$(el).next()` | `el.nextElementSibling` |
| `$(el).parents('ul').first()` | `el.closest('ul')` |
| `$(el).each(fn)` | `Array.from(els).forEach(fn)` or `els.forEach(fn)` (for NodeList) |
| `$('#x').click(fn)` | `document.getElementById('x').addEventListener('click', fn)` |
| `$(document).on('click', '.x', fn)` | document-level delegation (see §6.3) |
| `$.param(obj)` | `new URLSearchParams(obj).toString()` |
| `$('form').serialize()` | `new URLSearchParams(new FormData(form)).toString()` |
| `$('form').serializeArray()` | `Object.fromEntries(new FormData(form))` |

### 6.3 Document-level delegation

Replace `$(document).on('click', '.foo', fn)` with:

```js
document.addEventListener('click', function(e) {
    var btn = e.target.closest('.foo');
    if (!btn) return;
    e.preventDefault();
    // ...
});
```

This survives AJAX content swaps automatically.

### 6.4 Bootstrap 5 component access

```js
// Tooltips
document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(function(el) {
    new bootstrap.Tooltip(el);
});

// Modals
var modalEl = document.getElementById('myModal');
var modal = bootstrap.Modal.getOrCreateInstance(modalEl);
modal.show();   // .hide() / .toggle()

// Dropdowns
var ddEl = document.getElementById('myDropdownBtn');
var dd = bootstrap.Dropdown.getOrCreateInstance(ddEl);
dd.toggle();

// Collapse
var cEl = document.getElementById('myCollapse');
var collapse = bootstrap.Collapse.getOrCreateInstance(cEl, { toggle: false });
collapse.show();   // .hide() / .toggle()

// Tabs
var tabEl = document.querySelector('button[data-bs-target="#tab-reply"]');
var tab = bootstrap.Tab.getOrCreateInstance(tabEl);
tab.show();
```

### 6.5 Paradigm global helpers (replacing jQuery plugin form)

`assets/scripts/admin/paradigm/application.js` exposes these globals on `window`. They replace the jQuery plugin form (`$.fn.blestaXxx`):

| v5 (jQuery plugin) | v6 (vanilla global) |
|---|---|
| `$('a').blestaModal({…})` | `blestaModal(element, {…})` |
| `$('a').blestaModalConfirm({…})` | `blestaModalConfirm(element, {…})` (or declarative `.modal-confirm-delete` class) |
| `$('.col').blestaLoadWidgets({…})` | `blestaLoadWidgets('.col', {…})` |
| `$.blestaRequest(method, uri, data, …)` | `blestaRequest(method, uri, data, …)` |
| `$('input').blestaBindToolTips()` | `blestaBindToolTips(container)` |
| `$('textarea').blestaBindWysiwygEditor({…})` | `blestaBindWysiwygEditor(selector, {…})` |
| (legacy markdown editor) | `blestaBindMarkdownEditor(container)` (Toast UI) |
| `$('.drag').blestaSortable(handle, {…})` | `blestaSortable(element, handle, {…})` (Sortable.js wrapper) |
| `$('table').blestaUpdateRow(uri, '.x')` | `blestaUpdateRow(element, uri, container)` |
| `$('a').blestaTabbedContent()` | use native BS5 tabs (`data-bs-toggle="tab"`) |
| `$('.area').blestaBindToggleEvent(…)` | use BS5 Collapse (`data-bs-toggle="collapse"`) |
| `$('input').blestaBindPasswordGenerator()` | `blestaBindPasswordGenerator(container)` |

Other useful globals: `blestaBindGuiEvents(container, options)` (master init), `blestaBindGlobalGuiEvents(container, options)`, `blestaBindDatePicker(container)`, `blestaBindDropZone(container)`, `blestaSubmitOnClick(element, formId)`, `blestaScrollDom`, `blestaSetInvTotals`, `blestaPushState`, `blestaSlideToggle`, `blestaBindCloseMessage`, `blestaBindPreventDoubleSubmission`, `blestaDisableFormSubmission`, `blestaEnableFormSubmission`.

### 6.6 The `blestaAjaxComplete` event

After AJAX-swapping content into the DOM, dispatch this event so global initializers (expandable rows, quick-jump menu, tooltips, modal-confirm bindings) re-attach:

```js
document.dispatchEvent(new Event('blestaAjaxComplete'));
```

Listeners include `expandable-rows.js`, the quick-jump dropdown, and various view-level widgets. **You must dispatch this** when you replace a chunk of DOM with `innerHTML = …`.

### 6.7 Modal confirm pattern (declarative)

Replaces `$.fn.blestaModalConfirm`:

```html
<button type="button" class="btn btn-sm btn-outline-danger modal-confirm-delete"
        data-confirm-message="<?php echo $this->Html->safe($confirm_msg);?>"
        title="Delete">
    <i class="bi bi-trash"></i>
</button>
```

Three button classes set color/icon: `.modal-confirm` (neutral), `.modal-confirm-success` (green), `.modal-confirm-delete` (red). They are auto-bound by Paradigm globals. After AJAX swaps, re-bind explicitly:

```js
function initModalConfirms() {
    if (typeof blestaModalConfirm === 'function') {
        blestaModalConfirm('.modal-confirm');
        blestaModalConfirm('.modal-confirm-delete');
        blestaModalConfirm('.modal-confirm-success');
    }
}
document.addEventListener('blestaAjaxComplete', initModalConfirms);
```

When the trigger is inside a `<form>` (delete is a real form submit, not just a confirm dialog) wrap in `Form->create([...], ['class' => 'd-inline'])` so the form sits inline.

### 6.8 Sortable (`Sortable.js`)

```js
// jQuery UI sortable (legacy)
$('table.fields_table tbody').blestaSortable(false, {
    cancel: 'tr.expand_details,input,textarea',
    start: function() { ... },
    stop: function() { ... }
});

// Sortable.js (Paradigm)
var tbody = document.querySelector('table.fields_table tbody');
if (tbody && typeof Sortable !== 'undefined') {
    Sortable.create(tbody, {
        animation: 150,
        handle: '.sortable-drag-handle',     // or .movable
        draggable: 'tr[data-field-id]',
        ghostClass: 'sortable-ghost',
        onEnd: function(evt) {
            // POST new order to server with CSRF token
        }
    });
}
```

### 6.9 Toast UI Editor access

```js
// Get the editor instance from a textarea with data-markdown-editor
var textarea = document.getElementById('reply_details');
var editorDiv = textarea ? textarea.nextElementSibling : null;
while (editorDiv && !editorDiv._editor) {
    editorDiv = editorDiv.nextElementSibling;
}
if (editorDiv && editorDiv._editor) {
    var editor = editorDiv._editor;
    editor.changeMode('wysiwyg');           // or 'markdown'
    editor.changePreviewStyle('vertical');  // or 'tab'
    editor.insertText('text\n');
    editor.setMarkdown(currentContent);
    editor.focus();
}
```

The editor is bound by the global `blestaBindMarkdownEditor()` which scans for `textarea[data-markdown-editor]` on DOMContentLoaded. To get inline image upload support, set `data-image-upload-url`:

```php
$this->Form->fieldTextarea('details', $val, [
    'id' => 'reply_details',
    'rows' => 21,
    'data-markdown-editor' => '',
    'data-image-upload-url' => $this->Html->safe($this->base_uri . 'plugin/your_plugin/yourcontroller/uploadimage/')
]);
```

### 6.10 Submit-button FormData gotcha

`new FormData(form)` does NOT include the value of the submit button that was clicked. If your code submits programmatically and you need the submit button name/value, append manually:

```js
var formData = new FormData(form);
formData.append(submitBtn.name, submitBtn.value);
```

### 6.11 CSRF tokens

Every AJAX POST needs `_csrf_token`. Get it from `Form->getCsrfToken()` (no args) and emit it from PHP into the script, or read it from a hidden input:

```js
var csrfInput = document.querySelector('input[name="_csrf_token"]');
var csrfToken = csrfInput ? csrfInput.value : '<?php echo $this->Form->getCsrfToken();?>';

var formData = new FormData();
formData.append('_csrf_token', csrfToken);
formData.append('foo', 'bar');

fetch(url, {
    method: 'POST',
    body: formData,
    headers: { 'X-Requested-With': 'XMLHttpRequest' }
})
.then(r => r.json())
.then(data => { /* … */ });
```

### 6.12 Locale-safe interpolation in JS strings

When a translated string is placed inside a JS string literal, run it through `Html->safe` to escape quotes:

```diff
- var text = '<?php echo $this->_('Foo.text', true);?>';
+ var text = '<?php echo $this->Html->safe($this->_('Foo.text', true));?>';
```

### 6.13 Including plugin JS files

For ad-hoc plugin JS (non-Gulp-compiled), include via direct `<script>` tag — **not** `Javascript->setFile()`:

```php
<script src="<?php echo $this->view_dir;?>js/your-plugin.js"></script>
```

`Javascript->setFile()` prepends a `default_path` that double-prefixes the already-complete `$this->view_dir` and produces broken URLs.

A plugin's `views/default/js/` directory is **not Gulp-compiled** — it's served as-is from the plugin views directory.

---

## 7. HTML / markup structural changes

### 7.1 Form + Widget ordering reversed

In v5, Widget opens before Form. In v6, **Form opens first, Widget closes first**. This lets the submit button live in the card-footer while still belonging to the form.

```diff
- $this->Widget->create($title);
- $this->Form->create($url);
-     <!-- content + submit -->
- $this->Form->end();
- $this->Widget->end();
+ $this->Form->create($url);
+ $this->Widget->create($title);
+     <!-- content -->
+     $this->Widget->footer();
+     <!-- submit buttons in card-footer -->
+ $this->Widget->end();
+ $this->Form->end();
```

### 7.2 Card body / content / footer

`<div class="inner">`, `<div class="pad">`, `<div class="title_row">`, `<div class="button_row">` are all gone. Replace with BS5 card sections:

```diff
- $this->Widget->create('Title');
- ?>
- <div class="inner">
-     <div class="title_row first"><h3>Section Heading</h3></div>
-     <div class="pad">
-         <ul>
-             <li>... fields ...</li>
-         </ul>
-     </div>
-     <div class="button_row">
-         <?php $this->Form->fieldSubmit('save', 'Save', ['class' => 'btn btn-primary float-right']);?>
-     </div>
- </div>
- <?php $this->Widget->end(); ?>

+ <?php $this->Form->create(null, ['class' => 'disable-on-submit']); ?>
+ <?php $this->Widget->setBodyWrapper(false);   // listing pages only — see §9 ?>
+ <?php $this->Widget->create('Title'); ?>
+     <h6 class="form-section-header">Section Heading</h6>
+     <div class="mb-3"> ... fields ... </div>
+     <?php $this->Widget->footer(); ?>
+     <div class="d-flex justify-content-end gap-2">
+         <a href="<?php echo $cancel_url;?>" class="btn btn-outline-secondary">Cancel</a>
+         <?php $this->Form->fieldSubmit('save', 'Save', ['class' => 'btn btn-primary']);?>
+     </div>
+ <?php $this->Widget->end(); ?>
+ <?php $this->Form->end(); ?>
```

If you suppress the body wrapper with `setBodyWrapper(false)`, you provide your own `<div class="card-body">` (or run flush — useful for tables).

### 7.3 Tables

```diff
- <table class="table" id="ticket_list">
-     <tr class="heading_row">
-         <td>Heading</td>
-         <td class="last">Last</td>
-     </tr>
-     <?php foreach ($items as $i => $item) { ?>
-     <tr class="<?php echo ($i%2==1) ? 'odd_row' : '';?>">
-         <td>...</td>
-         <td class="last">...</td>
-     </tr>
-     <?php } ?>
- </table>
+ <div class="table-responsive">
+ <table class="table table-hover mb-0" id="ticket_list">
+     <thead>
+         <tr>
+             <th>Heading</th>
+             <th>Last</th>
+         </tr>
+     </thead>
+     <tbody>
+         <?php foreach ($items as $item) { ?>
+         <tr>
+             <td>...</td>
+             <td>...</td>
+         </tr>
+         <?php } ?>
+     </tbody>
+ </table>
+ </div>
```

Drop manual zebra striping (`odd_row`) — `.table-hover` and `.table-striped` handle row backgrounds.

### 7.4 Sortable column headers

```diff
- <td><span><a href="?sort=code" class="ajax<?php echo $sort=='code' ? ' '.$order : '';?>">Code</a></span></td>
+ <th>
+     <a href="?sort=code" class="ajax text-decoration-none">
+         Code
+         <?php if ($sort == 'code') { ?>
+             <i class="bi bi-chevron-<?php echo $order == 'asc' ? 'up' : 'down';?>"></i>
+         <?php } ?>
+     </a>
+ </th>
```

### 7.5 Expandable rows (Paradigm pattern)

Replace v5's `<tr class="expand_details" id="orders_123"><td class="subtable">` lazy-load pattern with `data-collapse-row` on a trigger row + `<tr class="collapse">` sibling:

```html
<tr class="order-row" data-collapse-row data-order-id="123" aria-expanded="false" style="cursor: pointer;">
    <td>...</td>
</tr>
<tr class="collapse">
    <td colspan="7" class="order-details-cell" style="padding: 0; background-color: var(--bg-secondary);">
        <div class="order-details-content p-3" data-order-id="123">
            <div class="text-center py-3">
                <div class="spinner-border spinner-border-sm" role="status">
                    <span class="visually-hidden">Loading...</span>
                </div>
            </div>
        </div>
    </td>
</tr>
```

Auto-wired by `assets/scripts/admin/paradigm/expandable-rows.js`. Lazy-load on `show.bs.collapse`:

```js
document.addEventListener('show.bs.collapse', function(e) {
    if (!e.target.classList.contains('collapse')) return;
    var container = e.target.querySelector('.order-details-content');
    if (!container || container.hasAttribute('data-loaded')) return;
    var orderId = container.getAttribute('data-order-id');
    fetch('<?php echo $this->base_uri;?>plugin/order/admin_main/orderdetails/' + orderId, {
        headers: { 'X-Requested-With': 'XMLHttpRequest' }
    })
    .then(r => r.text())
    .then(html => {
        container.innerHTML = html;
        container.setAttribute('data-loaded', 'true');
        document.dispatchEvent(new Event('blestaAjaxComplete'));
    });
});
```

Interactive children (`a`, `button`, `input`, `select`, `textarea`, `label`) are auto-excluded from the row's click handler — clicking a link inside the row won't toggle the collapse.

### 7.6 Dropdowns

```diff
- <a class="dropdown-toggle" data-toggle="dropdown" href="#">Menu</a>
- <ul class="dropdown-menu"><li><a>Item</a></li></ul>
+ <button type="button" class="btn btn-outline-secondary btn-sm dropdown-toggle"
+         data-bs-toggle="dropdown" data-bs-auto-close="outside" aria-expanded="false">
+     Menu
+ </button>
+ <ul class="dropdown-menu">
+     <li><a class="dropdown-item" href="#">Item</a></li>
+ </ul>
```

`<a class="dropdown-item">` is required for menu items (BS5 styles target this class). For dropdowns containing a search box, use `data-bs-auto-close="outside"` so clicking the input doesn't close the menu.

### 7.7 Accordions

```html
<div class="accordion accordion-flush" id="myAccordion">
    <div class="accordion-item border-0 bg-transparent">
        <h2 class="accordion-header">
            <button class="accordion-button bg-transparent px-0 py-1<?php echo $expanded ? '' : ' collapsed';?>"
                    type="button" data-bs-toggle="collapse" data-bs-target="#myCollapse"
                    aria-expanded="<?php echo $expanded ? 'true' : 'false';?>">
                Title
            </button>
        </h2>
        <div id="myCollapse" class="accordion-collapse collapse<?php echo $expanded ? ' show' : '';?>"
             data-bs-parent="#myAccordion">
            <div class="accordion-body p-0 pt-2">...</div>
        </div>
    </div>
</div>
```

### 7.8 Alert boxes

```diff
- <div class="accent_box">
-     <code>{tag1}</code> <code>{tag2}</code>
- </div>
+ <div class="alert alert-info">
+     <h6 class="alert-heading mb-2"><i class="bi bi-tags me-2"></i>Tags</h6>
+     <div class="d-flex flex-wrap gap-2">
+         <code>{tag1}</code>
+         <code>{tag2}</code>
+     </div>
+ </div>
```

For alerts with icons:

```html
<div class="alert alert-warning d-flex align-items-start" role="alert">
    <i class="bi bi-shield-exclamation me-2 mt-1"></i>
    <div class="flex-grow-1">
        <strong>Heads up!</strong>
        <pre class="small mt-2 mb-0 p-2 bg-body-secondary rounded"><code>example</code></pre>
    </div>
</div>
```

### 7.9 File upload zone

```html
<div class="file-upload-compact mb-3" id="fileDropZone">
    <?php $this->Form->fieldFile('attachment[]', [
        'id' => 'fileInput',
        'class' => 'drag-drop',
        'rel' => $this->_('AppController.dropzone.text', true),
        'style' => 'display:none;',
        'multiple' => 'multiple'
    ]);?>
    <div class="d-flex align-items-center gap-3">
        <i class="bi bi-cloud-upload upload-icon"></i>
        <div class="flex-grow-1"><div class="upload-text">Drop files here</div></div>
        <button type="button" class="btn btn-outline-primary btn-sm"
                onclick="document.getElementById('fileInput').click()">Browse</button>
    </div>
</div>
<div id="fileList" class="mb-3"></div>
```

### 7.10 Dual-select / drag-and-drop list

Replaces the v5 multi-select pair. Uses HTML5 drag-and-drop with a hidden `<select multiple>` so the form still submits the same field name.

```html
<div class="dual-select-container mb-4">
    <div class="dual-select-panel">
        <label class="form-label">Assigned</label>
        <div class="dual-select-dropzone">
            <div class="drag-list" id="assigned_departments">
                <div class="drag-item" data-value="1" draggable="true">Sales</div>
            </div>
        </div>
        <?php $this->Form->fieldMultiSelect('departments[]', [], [], [
            'id' => 'assigned', 'style' => 'display:none;'
        ]);?>
    </div>
    <div class="dual-select-controls">
        <button type="button" class="btn btn-sm btn-outline-primary dual-select-btn move_left">
            <i class="bi bi-arrow-left"></i>
        </button>
        <button type="button" class="btn btn-sm btn-outline-primary dual-select-btn move_right">
            <i class="bi bi-arrow-right"></i>
        </button>
    </div>
    <div class="dual-select-panel">
        <label class="form-label">Available</label>
        <div class="dual-select-dropzone">
            <div class="drag-list" id="available_departments">...</div>
        </div>
    </div>
</div>
```

### 7.11 Card grids (folders / responses)

```html
<div class="row g-3">
    <div class="col-12 col-sm-6 col-md-4 col-xl-3">
        <div class="response-folder-card" role="link" tabindex="0" data-href="<?php echo $url;?>">
            <div class="response-folder-icon"><i class="bi bi-folder-fill"></i></div>
            <div class="response-folder-name"><?php echo $cat->name;?></div>
            <span class="response-folder-count badge bg-secondary"><?php echo $cat->item_count;?></span>
            <div class="response-folder-actions">
                <a href="<?php echo $edit_url;?>" class="response-folder-action-btn"><i class="bi bi-pencil"></i></a>
                <button type="button" class="response-folder-action-btn modal-confirm-delete"
                        data-confirm-message="<?php echo $this->Html->safe($confirm);?>">
                    <i class="bi bi-trash"></i>
                </button>
            </div>
        </div>
    </div>
</div>
```

JS makes the card behave as a link via `data-href` + `click`/`Enter` keydown.

### 7.12 No more `is_ajax` wrappers

Remove the legacy `<?php if (!$is_ajax) { ?> <div id="right_outer"><section id="right_container"> <?php } ?>` scaffolding and matching close. The two-column layout is owned by Paradigm's structure now (see §10). Plugin views emit content only.

---

## 8. Form helper usage

### 8.1 Required class injection

The `Form` helper does NOT auto-inject BS5 classes. On every call, pass:

| Helper | Required class |
|---|---|
| `fieldText`, `fieldPassword`, `fieldFile`, `fieldTextarea` (non-markdown), `fieldHidden` (no class needed), `fieldUrl`, `fieldEmail`, `fieldTel`, `fieldNumber`, `fieldDate` | `'class' => 'form-control'` |
| `fieldSelect`, `fieldMultiSelect` | `'class' => 'form-select'` |
| `fieldCheckbox`, `fieldRadio` | `'class' => 'form-check-input'` |
| `Form->label` (block) | `'class' => 'form-label'` |
| `Form->label` (next to checkbox/radio) | `'class' => 'form-check-label'` |
| `fieldSubmit` | `'class' => 'btn btn-primary'` (or appropriate) |

### 8.2 ID conventions for bracketed names

Replace `[]` with `_` in IDs because `<label for="">` requires a valid HTML id:

```diff
- 'id' => 'settings[ticket_emails][' . $priority . ']'
+ 'id' => 'ticket_emails_' . $priority
```

The form field `name` keeps the brackets — only the `id` changes.

### 8.3 The `form` attribute (sidebar inputs)

When form inputs live in a separately-rendered sidebar partial (not inside the `<form>`), each must reference the form by id:

```php
$this->Form->fieldSelect('department_id', $departments, $val, [
    'id' => 'department_id',
    'class' => 'form-select form-select-sm',
    'form' => 'create_ticket'   // ties this input to <form id="create_ticket">
]);
```

### 8.4 Inline forms

Wrap delete-button forms with `'class' => 'd-inline'` so they sit inline with adjacent buttons:

```diff
- $this->Form->create($delete_url);
+ $this->Form->create($delete_url, ['class' => 'd-inline']);
```

### 8.5 Section headers

```diff
- <div class="title_row first"><h3>Section</h3></div>
+ <h6 class="form-section-header">Section</h6>
```

`form-section-header` is a Paradigm helper class.

### 8.6 Hint text below fields

```html
<div class="form-text">
    <i class="bi bi-info-circle"></i> Helpful description of the field.
</div>
```

### 8.7 Tooltip indicator

```html
<i class="bi bi-info-circle ms-1"
   data-bs-toggle="tooltip" data-bs-placement="top"
   title="<?php echo $this->Html->safe($this->_('Foo.tooltip.text', true));?>"></i>
```

The tooltip text moves from a `<div>` body of the legacy `<span class="tooltip">…</span>` into the `title` attribute.

### 8.8 Cancel/save button row

```html
<div class="d-flex justify-content-end gap-2">
    <a href="<?php echo $cancel_url;?>" class="btn btn-outline-secondary">Cancel</a>
    <?php $this->Form->fieldSubmit('save', 'Save', ['class' => 'btn btn-primary']);?>
</div>
```

Drop `pull-right`/`float-end` from the submit class — the wrapper handles alignment.

### 8.9 Radio button groups (BS5 button-group style)

```html
<div class="btn-group" role="group" aria-label="Status">
    <input type="radio" class="btn-check" name="status" id="status_active" value="active" checked>
    <label class="btn btn-outline-success" for="status_active">
        <i class="bi bi-check-circle me-1"></i>Active
    </label>
    <input type="radio" class="btn-check" name="status" id="status_inactive" value="inactive">
    <label class="btn btn-outline-danger" for="status_inactive">
        <i class="bi bi-x-circle me-1"></i>Inactive
    </label>
</div>
```

---

## 9. Widget helper changes

### 9.1 New methods

| Method | Purpose |
|---|---|
| `setBodyWrapper(bool $wrap)` | Suppress the default `<div class="card-body">` wrapper. Use for tables that should run flush. |
| `setNavStyle(string $style)` | Override default `nav-tabs-box`. Common: `nav-tabs-custom`. |
| `footer()` | Close the card-body and open a `<div class="card-footer">` for buttons / pagination. |

### 9.2 `setWidgetButton` keyword changes

| v5 keyword | v6 keyword | Icon |
|---|---|---|
| `arrow` | `arrow` | `bi bi-arrows-collapse` / `bi bi-arrows-expand` |
| `setting` | `setting` | `bi bi-gear-fill` |
| `filter-toggle` | `filter-toggle` | `bi bi-funnel` / `bi bi-funnel-fill` |
| `full_screen` | **`full-screen`** (renamed: underscore → hyphen) | `bi bi-arrows-angle-contract` / `bi bi-arrows-angle-expand` |
| (none) | **`widget-width`** (NEW) | `bi bi-arrows-angle-contract` / `bi bi-arrows-angle-expand` |

`widget-width` toggles a dashboard widget between column width and full-width (`<section class="card" data-width="full">`).

```diff
- $this->Widget->setWidgetButton('full_screen');
+ $this->Widget->setWidgetButton('full-screen');

  // Optional: add expandable-width to dashboard widgets
+ $this->Widget->setWidgetButton('widget-width');
+ $this->Widget->setWidgetButton('arrow');
```

### 9.3 Icon keys in arrays

```diff
  $this->Widget->setLinkButtons([
      [
          'name' => 'Add',
          'attributes' => ['href' => $url],
-         'icon' => 'fas fa-plus'
+         'icon' => 'bi bi-plus-lg'
      ],
      [
          'name' => 'Back',
          'attributes' => ['href' => $back_url],
-         'icon' => 'fas fa-arrow-left'
+         'icon' => 'bi bi-arrow-left me-1'
      ]
  ]);
```

### 9.4 Tab attribute keys

When using `setLinks` for tabs, switch to `data-bs-` attributes:

```diff
  $this->Widget->setLinks([
      [
          'name' => 'Basic',
          'attributes' => [
              'id' => 'tab-basic',
-             'data-toggle' => 'tab',
-             'data-target' => '#tab_basic',
+             'data-bs-toggle' => 'tab',
+             'data-bs-target' => '#tab_basic',
              'href' => '#'
          ]
      ]
  ]);
```

Wrap tab panes:

```html
<div class="tab-content">
    <div class="tab-pane fade show active" id="tab_basic" role="tabpanel">…</div>
    <div class="tab-pane fade" id="tab_advanced" role="tabpanel">…</div>
</div>
```

### 9.5 Widget HTML output (so AJAX swap targets are right)

V6 emits:

```html
<section class="card" id="..." data-width="full">
  <div class="card-header d-flex justify-content-between align-items-center">…</div>
  <div class="card-content">
    {nav}
    {filters}
    <div class="card-body">…content…</div>
    [optional <div class="card-footer">…</div>]
  </div>
</section>
```

**The AJAX replacer is now `.card-content`** (was `.common_box_content` for admin in v5). If your code emits `{replacer: '.common_box_content', content: '…'}`, change it to `{replacer: '.card-content', content: '…'}`.

### 9.6 Widget pagination placement

```diff
- $this->Pagination->build();
- $this->Widget->end();

+ if (isset($this->Pagination) && $this->Pagination->hasPages()) {
+     $this->Widget->footer();
+     $this->Pagination->build();
+ }
+ $this->Widget->end();
```

### 9.7 setStyleSheet with a Paradigm-specific stylesheet

When you maintain both a v5-compatible CSS and a v6 Paradigm CSS, use a separate file for each:

```php
// Old (kept for legacy default theme)
$this->Widget->setStyleSheet($this->view_dir . 'css/styles.css', ['id' => 'plugin_styles']);

// New (Paradigm)
$this->Widget->setStyleSheet($this->view_dir . 'css/paradigm.css', ['id' => 'plugin_styles']);
```

---

## 10. The new sidebar pattern (Paradigm)

Paradigm renders a two-column shell when the controller publishes a side panel. Use this for ticket detail forms, package settings, etc., that benefit from a sidebar of meta fields.

### 10.1 Controller side

```php
// In your action method:
$sidebar_html = $this->partial('admin_yourcontroller_sidebar', [
    'vars'        => $vars,
    'options'     => $options,
    // ... whatever the partial needs
]);

$this->structure->set('side_bar_content', $sidebar_html);
$this->structure->set('side_bar', ['partials/plugin_sidebar', null, 'side-content-mobile-visible']);
```

The third element of `side_bar` is an optional layout class.

### 10.2 Partial shell (already in core)

`app/views/admin/paradigm/partials/plugin_sidebar.pdt`:

```php
<?php /** @var View $this **/ ?>
<?php echo $side_bar_content ?? '';?>
```

A thin pass-through.

### 10.3 Plugin sidebar partial conventions

```php
<?php /** @var View $this **/ ?>
<div class="side-content-section">
    <h6 class="side-content-title">
        <i class="bi bi-info-circle"></i>
        <?php $this->_('AdminYour.heading_details');?>
    </h6>

    <div class="mb-3">
        <label class="form-label small text-secondary mb-1">Department</label>
        <?php $this->Form->fieldSelect('department_id', $departments, $vars->department_id ?? null, [
            'id' => 'department_id',
            'class' => 'form-select form-select-sm',
            'form' => 'create_ticket'   // tied to main form by id
        ]);?>
    </div>

    <div class="d-grid gap-2">
        <button type="submit" class="btn btn-primary btn-sm" form="create_ticket">Save</button>
    </div>
</div>
```

Conventions:

- Each grouped block: `<div class="side-content-section">`
- Section title: `<h6 class="side-content-title">` with leading `<i class="bi bi-…">`
- Field labels: `<label class="form-label small text-secondary mb-1">`
- Inputs use `-sm` size variants
- Inputs that should submit with a form elsewhere on the page: add `'form' => 'main_form_id'`
- Full-width stacked buttons: `<div class="d-grid gap-2">`

### 10.4 Sidebar JS communication

For sidebar inputs that submit a separate filter form, use document-level event delegation:

```js
document.addEventListener('submit', function(e) {
    if (e.target.id === 'ticket_filters') {
        e.preventDefault();
        applyTicketFilters();
    }
});
```

Inputs added by your sidebar partial appear in the parent form's submission automatically because of the `form` attribute.

---

## 11. CSS, SASS, and theme variables

### 11.1 Plugin CSS location

Same as v5: `plugins/{plugin}/views/default/css/styles.css`. Register via Widget:

```php
$this->Widget->setStyleSheet($this->view_dir . 'css/styles.css', ['id' => 'your_plugin_styles']);
```

Plugin CSS is **not Gulp-compiled** — it's served as-is.

### 11.2 CSS variables (theme tokens)

Paradigm exposes its own CSS custom properties. **Use these — do NOT hard-code colors** so your styles work in light AND dark mode.

```css
/* Paradigm tokens (NOT Bootstrap's --bs-* — those exist too but Paradigm prefers these) */
var(--bg-primary)
var(--bg-secondary)        /* common for striped / inset backgrounds */
var(--bg-tertiary)
var(--bg-card)
var(--border-color)         /* common for dividers */
var(--text-primary)
var(--text-secondary)
var(--text-muted)
var(--accent-color)
var(--link-color)
var(--success-color)
var(--warning-color)
var(--danger-color)

/* Some --bs-* overrides set by Paradigm (use these too) */
var(--bs-modal-bg)
var(--bs-modal-color)
var(--bs-modal-border-color)
var(--bs-body-bg)
var(--bs-border-color-translucent)
```

Active theme is set via `<html data-bs-theme="dark|light">` (also `data-theme`). Test both modes.

```css
/* Good */
.your-element {
    background-color: var(--bg-secondary);
    color: var(--text-primary);
    border: 1px solid var(--border-color);
}

/* Bad — breaks in dark mode */
.your-element {
    background-color: #f8f9fa;
    color: #212529;
    border: 1px solid #dee2e6;
}
```

### 11.3 Theme-aware Bootstrap utilities

Prefer these over hard-coded BS classes:

| Replace | With |
|---|---|
| `bg-light` | `bg-body-secondary` |
| `bg-white` | `bg-body` |
| `text-dark` (when meaning "default body color") | `text-body` |
| `text-secondary` | `text-secondary` (works) |
| `border` (default border color) | works (uses `--bs-border-color`) |

### 11.4 Code editors that respond to theme

For embedded code editors (Ace, Monaco, etc.), watch for theme changes:

```js
function getAceTheme() {
    return document.documentElement.getAttribute('data-bs-theme') === 'dark'
        ? 'ace/theme/tomorrow_night'
        : 'ace/theme/tomorrow';
}
new MutationObserver(function() {
    var theme = getAceTheme();
    Object.values(editors).forEach(function(e) { e.setTheme(theme); });
}).observe(document.documentElement, {
    attributes: true, attributeFilter: ['data-bs-theme']
});
```

### 11.5 SCSS backwards-compat shims

Paradigm's `_backwards.scss` re-styles many v5 class names (`pull-left`, `pull-right`, `common_box`, `inner`, `pad`, `title_row`, `button_row`, etc.) on top of BS5 so unconverted plugins look acceptable. **Don't rely on these for new code** — they exist only to soften the migration.

---

## 12. AJAX patterns

### 12.1 Replace `blestaRequest` with `fetch`

```diff
- $(document).blestaRequest('POST', url,
-     { _csrf_token: token, foo: 'bar' },
-     function(data) { /* success */ },
-     null,
-     { dataType: 'json' }
- );

+ var formData = new FormData();
+ formData.append('_csrf_token', token);
+ formData.append('foo', 'bar');
+
+ fetch(url, {
+     method: 'POST',
+     body: formData,
+     headers: { 'X-Requested-With': 'XMLHttpRequest' }
+ })
+ .then(r => r.json())
+ .then(data => { /* success */ })
+ .catch(err => console.error(err));
```

The global `blestaRequest()` (vanilla version) still exists if you prefer it.

### 12.2 The `X-Requested-With` header

It's required to trigger the controller's `renderAjaxWidgetIfAsync()` JSON-envelope wrapping. jQuery added it automatically; with `fetch` you must add it manually.

### 12.3 The JSON envelope

When the controller wraps a widget response, you get:

```json
{ "replacer": ".card-content", "content": "<div>…</div>" }
```

Handle both cases (replacer null = full content swap):

```js
fetch(url, { headers: { 'X-Requested-With': 'XMLHttpRequest' }})
.then(r => r.json())
.then(data => {
    var widget = document.getElementById('admin_tickets');
    if (data.replacer == null) {
        widget.innerHTML = data.content;
    } else {
        var target = widget.querySelector(data.replacer);
        if (target) target.innerHTML = data.content;
    }
    document.dispatchEvent(new Event('blestaAjaxComplete'));
});
```

### 12.4 New AJAX endpoints in plugin controllers

Plugin controllers ARE directly routable (unlike main app controllers). Adding `public function myAjaxEndpoint()` to a plugin controller exposes it at `/plugin/your_plugin/your_controller/myajaxendpoint/`. No `index()` switching needed.

### 12.5 `outputAsJson` for custom JSON responses

```php
public function myAjaxEndpoint()
{
    $result = ['ok' => true, 'data' => [...]];
    $this->outputAsJson($result);
    return false;
}
```

### 12.6 Lazy-load on collapse

```js
document.addEventListener('show.bs.collapse', function(e) {
    if (!e.target.classList.contains('collapse')) return;
    var container = e.target.querySelector('.your-content-container');
    if (!container || container.hasAttribute('data-loaded')) return;
    var id = container.getAttribute('data-record-id');
    fetch(url + id, { headers: { 'X-Requested-With': 'XMLHttpRequest' }})
        .then(r => r.text())
        .then(html => {
            container.innerHTML = html;
            container.setAttribute('data-loaded', 'true');
            document.dispatchEvent(new Event('blestaAjaxComplete'));
        });
});
```

---

## 13. Module-specific changes (PHP base classes)

### 13.1 Files removed

These v5 files are deleted in v6:

- `components/modules/module_field.php`
- `components/modules/module_fields.php`

The global short names `ModuleField` and `ModuleFields` are kept as **runtime class aliases** in `config/aliases.php`:

```php
'ModuleFields' => Blesta\Core\Util\Input\Fields\InputFields::class,
'ModuleField'  => Blesta\Core\Util\Input\Fields\InputField::class,
```

**No source change required** — `new ModuleFields()` and `new ModuleField(...)` still work. The canonical class is `Blesta\Core\Util\Input\Fields\InputFields`.

### 13.2 `Module` base class additions

#### New: `getIcon()`

Modules can declare a Bootstrap icon for Paradigm-native rendering by adding `"icon": "bi-…"` (name only — see the note below) to their `config.json`:

```json
{
    "version": "2.20.0",
    "name": "Cpanel.name",
    "description": "Cpanel.description",
    "icon": "bi-server",
    "authors": [ /* ... */ ],
    "module": { /* ... */ }
}
```

> The `config.json` `icon` value is the icon **name only** (`bi-server`, `bi-globe`), matching the shipped
> cpanel/namesilo. The admin renders it as `<i class="bi {icon}">`, prepending the `bi` base class — so
> `"bi bi-server"` would produce the broken `class="bi bi bi-server"`.

The base `Module::getIcon()` reads `$this->config->icon` and returns it (or `null`). Paradigm uses this in its module list / overview screens.

#### Renamed: `arrayToModuleFields()` → `arrayToInputFields()`

```diff
- protected function arrayToModuleFields($arr, ModuleFields $fields = null, $vars = null)
+ protected function arrayToInputFields($arr, InputFields $fields = null, $vars = null)
  {
      if (!$fields) {
-         $fields = new ModuleFields();
+         $fields = new InputFields();
      }
      // body unchanged
  }

  // Backwards-compatible alias kept (deprecated):
  protected function arrayToModuleFields($arr, InputFields $fields = null, $vars = null)
  {
      return $this->arrayToInputFields($arr, $fields, $vars);
  }
```

If your module overrides this, prefer `arrayToInputFields`. Old name still works.

The `@return` docblocks on `getPackageFields()`, `getAdminAddFields()`, `getClientAddFields()`, `getAdminEditFields()` updated `@return ModuleFields` → `@return InputFields` (runtime types are equivalent due to alias).

### 13.3 `RegistrarModule` base class

**Unchanged.** All public methods (`supportsDnsManagement`, `checkAvailability`, `getDomainContacts`, `setDomainNameservers`, `getTldPricing`, `getFilteredTldPricing`, `registerDomain`, `transferDomain`, `renewDomain`, `restoreDomain`, etc.) have identical signatures.

### 13.4 Module field rendering / form patterns

Field-rendering helpers (`getPackageFields`, `getAdminAddFields`, `getClientAddFields`, `getAdminEditFields`) remain unchanged. They return `Blesta\Core\Util\Input\Fields\InputFields` and the framework iterates and renders.

The shared `service_fields*.pdt` views still use `<div class="form-group">` and `class="inline"` checkbox labels — these need the BS5 conversion (§4.5–4.6).

### 13.5 Module `manage.pdt` shape

The legacy widget chrome (`title_row`, `pad`, `inner`, `heading_row`, `odd_row`, `empty_section`, `<a class="manage rel="...">`) survives in shipped core modules but should be converted in your own modules using §4 / §7 rules.

### 13.6 Tab icon strings in module PHP (ADMIN tabs only)

Convert the `'icon' => 'fas fa-…'` values in **admin** tab arrays (`getAdminTabs()` / `getAdminServiceTabs()`)
to Bootstrap Icons. **Leave client tab arrays (`getClientTabs()` / `getClientServiceTabs()`) on
`fas fa-…`** — they render in the 6.0 client portal (FontAwesome).

```diff
  public function getAdminTabs($package)     // ADMIN — convert
  {
      return [
-         'tabActions' => ['name' => Language::_('Mod.tab_actions', true), 'icon' => 'fas fa-cog'],
+         'tabActions' => ['name' => Language::_('Mod.tab_actions', true), 'icon' => 'bi bi-gear'],
      ];
  }

  public function getClientTabs($package)    // CLIENT — leave as FontAwesome for 6.0
  {
      return [
          'tabClientActions' => ['name' => Language::_('Mod.tab_actions', true), 'icon' => 'fas fa-cog'],
      ];
  }
```

### 13.7 Blesta-specific theme classes — keep vs replace (read carefully)

These are Blesta theme classes (**not** Bootstrap classes). The Paradigm backwards-compat SCSS shim still
styles many of them so **unconverted** markup keeps working — but that compatibility is *not* a reason to
emit them in new/converted code. Be unambiguous about which list a class is in:

- **REPLACE when writing or converting** (these have direct BS5 replacements covered in §4/§7 — do not
  carry them forward): `title_row`, `pad`, `inner`, `button_row`, `heading_row`, `odd_row`,
  `empty_section`, `empty_box`, `tab_content`, `tabs`, `inline`, `stretch`, `accent_box`, `quiet`, `block`,
  `clear`.
- **KEEP — genuinely Blesta-functional, no BS5 equivalent** (leave these alone): `manage`, `field_row`,
  `red_txt`, `buff`, `running`, `stopped`, `border_left`, `border_none`, `fixed_small`, `center`, `last`,
  and the behavioural hook class `ajax` (used on AJAX links/sort headers).

Rule of thumb for an AI converting a view: if a legacy class appears in §4 or §7 with a "before → after"
mapping, **replace it**. Otherwise, if it's in the KEEP list above, leave it. When unsure, prefer the
native BS5 utility/pattern over a legacy class.

---

## 14. Composer / PHP / dependency notes

### 14.1 PHP requirement

```diff
  "require": {
-     "php": ">=7.2.0",
+     "php": ">=8.2.0",
      ...
  }
```

Use modern PHP idioms in new code:

- Null coalescing: `$foo ?? null` (not `isset($foo) ? $foo : null`)
- Null safe: `$foo?->bar`
- Match expressions, named args, readonly props, enums, first-class callable syntax — all available

### 14.2 New top-level deps available to plugins

- **Illuminate (Laravel) Support, Collections, Str, Arr** — usable directly
- **HTMLPurifier** (`ezyang/htmlpurifier`)
- **Carbon** (via Illuminate)

Example:

```php
use Illuminate\Support\Str;

echo Str::of($description)->stripTags()->squish()->limit(140);
```

### 14.3 Your plugin's `config.json`

Bump `require.blesta` to `>=6.0.0` and your plugin version (typically a major bump). For modules, optionally add the `icon` key:

```json
{
    "version": "3.0.0",
    "name": "YourPlugin.name",
    "description": "YourPlugin.description",
    "icon": "bi bi-puzzle",
    "require": { "blesta": ">=6.0.0" },
    "authors": [ ... ]
}
```

### 14.4 Your `composer.json`

Standard skeleton remains:

```json
{
    "name": "your/plugin",
    "type": "blesta-plugin",   // or blesta-module / blesta-gateway
    "require": {
        "blesta/composer-installer": "~1.0"
    }
}
```

The `blesta/composer-installer` `type` keys (`blesta-plugin`, `blesta-module`, `blesta-gateway`, `blesta-invoice-format`) are unchanged.

---

## 15. Common gotchas

### 15.1 `new FormData(form)` omits submit button values

Manually append:

```js
formData.append(submitBtn.name, submitBtn.value);
```

### 15.2 PHPIDS scans every POST

Don't put base64-encoded binary data in form POST fields — PHPIDS's regex chokes on large strings and burns memory. Use file uploads or dedicated endpoints.

### 15.3 Plugin AJAX endpoints

Plugin controllers ARE directly routable: `public function myAjax()` on your plugin controller is reachable at `/plugin/your_plugin/your_controller/myajax/`. No `index()` switch needed. (Main-app controllers are different — there, custom action methods aren't routable and you must dispatch via a POST parameter inside `index()`.)

### 15.4 `setFile()` double-prefix bug

Don't use `$this->Javascript->setFile($this->view_dir . 'js/foo.js')`. The helper prepends a `default_path` that double-prefixes. Use a direct `<script>` tag:

```php
<script src="<?php echo $this->view_dir;?>js/foo.js"></script>
```

### 15.5 Tooltips don't auto-init

After page load and after every AJAX swap, run:

```js
document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(function(el) {
    new bootstrap.Tooltip(el);
});
```

Or rely on Paradigm's `blestaBindToolTips(container)` global.

### 15.6 Forgetting the `-bs-` infix

Easy to miss. Search-and-replace cycle:

```
data-toggle      → data-bs-toggle
data-target      → data-bs-target
data-dismiss     → data-bs-dismiss
data-parent      → data-bs-parent
data-placement   → data-bs-placement
data-html        → data-bs-html
data-container   → data-bs-container
data-auto-close  → data-bs-auto-close
```

### 15.7 `<select class="form-control">` looks wrong

In BS5 selects need `class="form-select"` to get the chevron caret styling. `form-control` still works mechanically but looks broken.

### 15.8 `<label class="inline">` next to checkboxes

Replace with `<label class="form-check-label">` and wrap both in `<div class="form-check">`.

### 15.9 Hard-coded colors break dark mode

Always use CSS variables (`var(--border-color)`, `var(--bg-secondary)`) or theme-aware utility classes (`bg-body-secondary`, not `bg-light`). Test in both light AND dark mode.

### 15.10 Deleting AJAX-bound content silently breaks event handlers

When you `innerHTML = …` a region of DOM, any event listeners attached to the replaced elements are gone. Either:

- Use document-level delegation (`document.addEventListener('click', e => { var btn = e.target.closest('.foo'); … })`)
- Re-bind after AJAX by listening to / dispatching `blestaAjaxComplete`

### 15.11 The `Pagination->build()` empty card-footer

Wrap pagination output in `if ($this->Pagination->hasPages()) { … }` so single-page lists don't render an empty card-footer with just a page-of-1 indicator.

### 15.12 Widget AJAX swap target changed

Existing v5 plugin endpoints that emitted `{replacer: '.common_box_content', content: '…'}` need to switch to `{replacer: '.card-content', content: '…'}` for admin. (Client portal already used `.card-content`.)

### 15.13 Module admin tabs render as "Array"

A **module's** `getAdminTabs()` / `getAdminServiceTabs()` must return plain strings (`'tabFoo' => "Title"`). Core uses the value directly as the tab label, so the `['name' => …, 'icon' => …]` array form renders the tab as the literal text "Array" and logs `E_WARNING: Array to string conversion` (`helpers/widget/widget.php`). The array form is only valid for **plugin** `getAdminServiceTabs()` and for client tabs (§5.2). Module admin tabs cannot carry icons.

### 15.14 Listing table doesn't span the full card width

If a table sits inset from the card edges, the widget's default `card-body` wrapper is padding it. Call `$this->Widget->setBodyWrapper(false)` before `Widget->create()` and give the empty-state branch its own `<div class="card-body text-center text-muted py-5">` (§9.1). Form views keep the default wrapper.

---

## 16. Canonical reference files

Use these v6 files as references when porting your own:

| Pattern | Reference file |
|---|---|
| List view with sortable headers, action buttons, pagination, modal-confirm-delete | `plugins/order/views/default/admin_forms.pdt` |
| Multi-tab settings form, BS5 button-radio groups, tooltip icons, template chooser, dual-select | `plugins/order/views/default/admin_forms_add.pdt` |
| Widget with category tabs (badges), expandable rows via Bootstrap Collapse, lazy-load on `show.bs.collapse`, bulk-actions bar | `plugins/order/views/default/admin_main.pdt` |
| Simple settings form with `card-body` + `card-footer` + cancel/save | `plugins/order/views/default/admin_settings.pdt` |
| Connected language-tab form with CKEditor 5 lazy-init per tab | `plugins/cms/views/default/admin_main_manage.pdt` |
| Status-conditional action buttons on rows | `plugins/order/views/default/admin_payouts.pdt` |
| Embed-codes alert + Ace editor theme-aware integration | `plugins/order/views/default/admin_forms_embed_codes.pdt` |
| Paradigm sidebar (collapsible groups) | `plugins/order/views/default/partials/admin_affiliates_sidebar.pdt` |
| Tooltip-wrapped row content using `data-bs-toggle="tooltip" data-bs-html="true"` | `plugins/order/views/default/admin_main_search.pdt` |
| Sidebar pattern (ticket meta in side panel) | `plugins/support_manager/views/default/admin_tickets_reply_sidebar.pdt` |
| Toast UI editor / ticket reply / inline image upload | `plugins/support_manager/views/default/admin_tickets_reply.pdt` |
| Ticket replies card pattern | `plugins/support_manager/views/default/admin_tickets_replies.pdt` |
| File upload zone + preview | `plugins/support_manager/views/default/admin_staff_edit.pdt` |
| Card grid (folders) | `plugins/support_manager/views/default/admin_responses_response_list.pdt` |
| Dual-select drag-drop list | `plugins/support_manager/views/default/admin_staff_add.pdt` |
| Modal confirm pattern (declarative) | `plugins/support_manager/views/default/admin_departments.pdt` |
| Widget feed with pagination | `plugins/feed_reader/views/default/admin_main.pdt` |
| Dashboard widget (`widget-width` button, ApexCharts, theme-aware filters) | `plugins/system_overview/views/default/admin_main.pdt` |

Core paradigm references:

| | |
|---|---|
| Main structure shell | `app/views/admin/paradigm/structure.pdt` |
| Sidebar partials | `app/views/admin/paradigm/partials/sidebar.pdt`, `…/plugin_sidebar.pdt`, `…/main_sidebar.pdt` |
| Theme manifest | `app/views/admin/paradigm/config.json` |
| Helper library JS (globals) | `assets/scripts/admin/paradigm/application.js` |
| Theme variables SCSS | `assets/sass/admin/paradigm/base/_variables.scss` |
| Backwards-compat shims | `assets/sass/admin/paradigm/_backwards.scss` |
| Expandable rows JS | `assets/scripts/admin/paradigm/expandable-rows.js` |
| Widget helper PHP | `public_html/helpers/widget/widget.php` |
| AbstractWidget base | `public_html/core/Util/Widgets/AbstractWidget.php` |
| AppController (asset/AJAX/CSRF) | `public_html/app/app_controller.php` |
| Module base class | `public_html/components/modules/module.php` |
| Class aliases (`ModuleFields`, etc.) | `public_html/config/aliases.php` |
| Gulp asset build | `gulpfile.js` |

---

## 17. LLM prompt template for one-shot conversion

Save a copy of this guide alongside your extension's source. Then feed it as context to your favorite model with a prompt like:

```
You are converting a Blesta 5.x plugin (or module) to be compatible with the
Blesta 6 Paradigm ADMIN theme (Bootstrap 5 + vanilla JS + Bootstrap Icons).

SCOPE — ADMIN SURFACES ONLY. Convert only admin .pdt files (e.g. manage.pdt,
add_row.pdt, edit_row.pdt, admin_service_info.pdt, admin tab views), admin
controllers/widgets, and inline <script> in those admin views. DO NOT touch
client-facing code: the Blesta 6.0 client portal is still Bootstrap 4 + jQuery +
FontAwesome. Specifically, LEAVE AS-IS: client_service_info.pdt, every
tab_client_*.pdt, getClient* method output, and getClientTabs/getClientServiceTabs
icon arrays (they keep 'fas fa-…'). If a file is client-facing, skip it.

Apply the rules in this guide exhaustively to the admin surfaces above:

1. Apply §2 conversion checklist line by line.
2. Apply §4 Bootstrap 4→5 class changes (data-bs- infix, float-end, ms/me,
   form-select, form-check, form-label, btn-secondary, badge bg-…, etc.).
3. Apply §5 icon mapping (every fas/far/fa fa-… becomes bi bi-…) in admin views
   and admin PHP icon arrays (setLinkButtons, setLinks, setWidgetButton, and
   PLUGIN getAdminServiceTabs). MODULE getAdminTabs/getAdminServiceTabs must
   return plain strings ('tabFoo' => "Title") — no arrays, no icons; an array
   renders the tab as the literal text "Array" (§5.2, §15.13). For config.json,
   set "icon": "bi-…" (NAME ONLY — the admin prepends the bi base class; do not
   write "bi bi-…"). Do NOT convert getClientTabs/getClientServiceTabs icons.
4. Apply §6 jQuery → vanilla JS conversion in admin views (no jQuery in admin).
   Replace $(document).ready, all selectors, all events, all $.fn.blestaXxx calls,
   blestaRequest, jQuery UI sortable, CKEditor binding. Use `fetch()` with
   X-Requested-With header for AJAX. (Client views may keep jQuery.)
5. Apply §7 markup changes (card-body/card-content/card-footer, table-responsive,
   table-hover, expandable rows via data-collapse-row, modal-confirm-delete buttons).
6. Apply §8 Form helper class injection (form-control, form-select, form-check-input,
   form-label, form-check-label) — every Form helper call must pass the right class.
7. Apply §9 Widget API changes (setBodyWrapper, footer(), full_screen → full-screen,
   widget-width, icon class strings in arrays).
8. Apply §11 theme variables — never hard-code colors. Use var(--border-color),
   var(--bg-secondary), bg-body-secondary, etc.
9. For modules, also apply §13: optional config.json icon key (name only), and
   rename arrayToModuleFields overrides to arrayToInputFields.
10. Test mentally in both light AND dark mode. Flag any hard-coded colors.

After each file, tell me what you changed and whether you skipped it as client-
facing. Don't add features or refactor anything beyond what the guide requires.
```

---

## Appendix A — Summary "before vs after" for a typical admin form

**Before (Blesta 5):**

```php
<?php
$this->Widget->create($this->_('AdminFoo.add.boxtitle', true));
?>
<div class="inner">
    <div class="title_row first"><h3><?php $this->_('AdminFoo.add.section');?></h3></div>
    <?php $this->Form->create($this->base_uri . 'plugin/your_plugin/admin_foo/add/'); ?>
    <div class="pad">
        <ul>
            <li>
                <?php
                $this->Form->label($this->_('AdminFoo.add.field_name', true), 'name');
                $this->Form->fieldText('name', $vars->name, ['id' => 'name']);
                ?>
            </li>
            <li>
                <?php
                $this->Form->label($this->_('AdminFoo.add.field_status', true), 'status');
                $this->Form->fieldSelect('status', $statuses, $vars->status, ['id' => 'status']);
                ?>
            </li>
            <li>
                <?php
                $this->Form->fieldCheckbox('enabled', '1', $vars->enabled == '1', ['id' => 'enabled']);
                $this->Form->label($this->_('AdminFoo.add.field_enabled', true), 'enabled', ['class' => 'inline']);
                ?>
            </li>
        </ul>
    </div>
    <div class="button_row">
        <a class="btn btn-default pull-right ajax" href="<?php echo $this->base_uri . 'plugin/your_plugin/admin_foo/';?>">
            <i class="fas fa-times"></i> <?php $this->_('AdminFoo.add.field_cancel');?>
        </a>
        <?php $this->Form->fieldSubmit('save', $this->_('AdminFoo.add.field_save', true),
            ['class' => 'btn btn-primary pull-right']); ?>
    </div>
    <?php $this->Form->end(); ?>
</div>
<?php $this->Widget->end(); ?>

<script type="text/javascript">
$(document).ready(function() {
    $('[data-toggle="tooltip"]').tooltip();
    $('#enabled').change(function() {
        $('.dependent-section').toggle(this.checked);
    });
});
</script>
```

**After (Blesta 6):**

```php
<?php /** @var View $this */ ?>
<?php $this->Form->create($this->base_uri . 'plugin/your_plugin/admin_foo/add/'); ?>
<?php $this->Widget->create($this->_('AdminFoo.add.boxtitle', true)); ?>
    <h6 class="form-section-header"><?php $this->_('AdminFoo.add.section');?></h6>

    <div class="mb-3">
        <?php
        $this->Form->label($this->_('AdminFoo.add.field_name', true), 'name', ['class' => 'form-label']);
        $this->Form->fieldText('name', $vars->name ?? '', ['id' => 'name', 'class' => 'form-control']);
        ?>
    </div>

    <div class="mb-3">
        <?php
        $this->Form->label($this->_('AdminFoo.add.field_status', true), 'status', ['class' => 'form-label']);
        $this->Form->fieldSelect('status', $statuses, $vars->status ?? null, ['id' => 'status', 'class' => 'form-select']);
        ?>
    </div>

    <div class="mb-3">
        <div class="form-check">
            <?php
            $this->Form->fieldCheckbox('enabled', '1', ($vars->enabled ?? '') == '1',
                ['id' => 'enabled', 'class' => 'form-check-input']);
            $this->Form->label($this->_('AdminFoo.add.field_enabled', true), 'enabled', ['class' => 'form-check-label']);
            ?>
        </div>
    </div>

    <?php $this->Widget->footer(); ?>
    <div class="d-flex justify-content-end gap-2">
        <a class="btn btn-outline-secondary ajax" href="<?php echo $this->base_uri . 'plugin/your_plugin/admin_foo/';?>">
            <?php $this->_('AdminFoo.add.field_cancel');?>
        </a>
        <?php $this->Form->fieldSubmit('save', $this->_('AdminFoo.add.field_save', true),
            ['class' => 'btn btn-primary']); ?>
    </div>
<?php $this->Widget->end(); ?>
<?php $this->Form->end(); ?>

<script type="text/javascript">
document.addEventListener('DOMContentLoaded', function() {
    document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(function(el) {
        new bootstrap.Tooltip(el);
    });

    var enabled = document.getElementById('enabled');
    if (enabled) {
        var sections = document.querySelectorAll('.dependent-section');
        var toggle = function() {
            sections.forEach(function(s) {
                s.style.display = enabled.checked ? '' : 'none';
            });
        };
        enabled.addEventListener('change', toggle);
        toggle();
    }
});
</script>
```

---

## Appendix B — Quick-reference search/replace table

Suitable for a regex-aware editor. Apply in this order, eyeballing each match:

| Find | Replace |
|---|---|
| `data-toggle=` | `data-bs-toggle=` |
| `data-target=` | `data-bs-target=` |
| `data-dismiss=` | `data-bs-dismiss=` |
| `data-parent=` | `data-bs-parent=` |
| `data-placement=` | `data-bs-placement=` |
| `data-html=` | `data-bs-html=` |
| `data-container=` | `data-bs-container=` |
| `\bpull-right\b` | `float-end` |
| `\bpull-left\b` | `float-start` |
| `\bfloat-right\b` | `float-end` |
| `\bfloat-left\b` | `float-start` |
| `\bml-(\d)\b` | `ms-$1` |
| `\bmr-(\d)\b` | `me-$1` |
| `\bpl-(\d)\b` | `ps-$1` |
| `\bpr-(\d)\b` | `pe-$1` |
| `\btext-left\b` | `text-start` |
| `\btext-right\b` | `text-end` |
| `\bfont-weight-bold\b` | `fw-bold` |
| `\bfont-weight-normal\b` | `fw-normal` |
| `\bsr-only\b` | `visually-hidden` |
| `\bbtn-default\b` | `btn-secondary` |
| `\bbtn-block\b` | `w-100` |
| `\bbadge-(primary\|secondary\|success\|danger\|warning\|info\|light\|dark)\b` | `bg-$1` |
| `\bform-group\b` | `mb-3` |
| `\bcustom-control custom-checkbox\b` | `form-check` |
| `\bcustom-control custom-radio\b` | `form-check` |
| `\bcustom-control-input\b` | `form-check-input` |
| `\bcustom-control-label\b` | `form-check-label` |
| `\bcustom-select\b` | `form-select` |
| `\bbg-light\b` | `bg-body-secondary` |
| `class="close"` | `class="btn-close"` |
| `<i class="fas fa-plus(\s\|")` | `<i class="bi bi-plus-lg$1` |
| `<i class="fas fa-pencil-alt(\s\|")` | `<i class="bi bi-pencil$1` |
| `<i class="fas fa-trash(\s\|")` | `<i class="bi bi-trash$1` |
| `<i class="fas fa-cog(\s\|")` | `<i class="bi bi-gear$1` |
| `<i class="fas fa-times(\s\|")` | `<i class="bi bi-x-lg$1` |
| `<i class="fas fa-eye(\s\|")` | `<i class="bi bi-eye$1` |
| `<i class="fas fa-question-circle(\s\|")` | `<i class="bi bi-question-circle$1` |
| `<i class="fas fa-info-circle(\s\|")` | `<i class="bi bi-info-circle$1` |
| `<i class="fas fa-power-off(\s\|")` | `<i class="bi bi-power$1` |
| `<i class="fas fa-sync-alt(\s\|")` | `<i class="bi bi-arrow-clockwise$1` |
| `<i class="fas fa-sign-in-alt(\s\|")` | `<i class="bi bi-box-arrow-in-right$1` |
| `<i class="fas fa-(\w[\w-]*)(\s\|")` | `<i class="bi bi-$1$2` (then verify each) |
| `\$\(document\)\.ready\(` | `document.addEventListener('DOMContentLoaded', ` |
| `\.blestaRequest\(` | (rewrite manually as `fetch`) |
| `'icon' => 'fas fa-(\w[\w-]*)'` | `'icon' => 'bi bi-$1'` |
| `setWidgetButton\('full_screen'\)` | `setWidgetButton('full-screen')` |

After bulk replacements, manually verify icon mappings against §5.1 (some FA names don't have a 1:1 BS Icons equivalent and need a different name).

---

*End of guide.*
