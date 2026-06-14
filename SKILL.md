---
name: extension-dev-kit
description: >-
  Author and port Blesta 6.0 extensions — modules (hosting/server and registrar),
  plugins, and payment gateways. Use when building a new Blesta extension from
  scratch or converting a 5.x extension to 6.0. Covers the PHP contract/method
  surface, config.json/manifest schema, data access, packaging, and the
  Bootstrap 5 / Paradigm UI conversion. Works as portable context for any AI
  tool; routes to per-type reference files on demand.
---

# Blesta 6 Extension Development

This skill helps you **create** new Blesta 6.0 extensions and **convert** existing 5.x
extensions. Blesta extensions come in three kinds:

- **Modules** — provision and manage services on remote systems. Two flavours:
  - *Hosting / server* modules (e.g. cPanel) — extend `Module`, no `type` in config.json.
  - *Registrar* modules (e.g. NameSilo) — extend `RegistrarModule`, **require** `"type": "registrar"`.
- **Plugins** — extend the application: events, cron tasks, admin/client actions, widgets, card/dashboard content.
- **Payment gateways** — process payments. Two flavours: `merchant` (card processors) and `nonmerchant` (off-site/redirect).

## How to use this skill

1. Identify the extension kind and (for modules) the flavour.
2. Load the relevant reference file(s) below. Do **not** load all of them — pull only what the task needs.
3. For any UI work (views, `.pdt` files), always also load `reference/common/conversion-ui.md`.

## Routing

| If you are… | Load |
|---|---|
| Building a **new module** | `reference/modules/create.md` |
| Porting a **module** from 5.x | `reference/modules/convert.md` + `reference/common/conversion-ui.md` |
| Porting a **module from WHMCS** (provisioning or registrar) | `reference/modules/convert-from-whmcs.md` + `reference/modules/create.md` |
| Building a **new plugin** | `reference/plugins/create.md` |
| Porting a **plugin** from 5.x | `reference/plugins/convert.md` + `reference/common/conversion-ui.md` |
| Porting a **WHMCS addon module** to a plugin | `reference/plugins/convert-from-whmcs.md` + `reference/plugins/create.md` |
| Building a **new gateway** | `reference/gateways/create.md` |
| Porting a **gateway** from 5.x | `reference/gateways/convert.md` + `reference/common/conversion-ui.md` |
| Porting a **gateway from WHMCS** (third-party or merchant) | `reference/gateways/convert-from-whmcs.md` + `reference/gateways/create.md` |
| Doing **any UI / view work** | `reference/common/conversion-ui.md` |

## Ground rules (apply to every extension type)

- **PHP ≥ 8.2.** Short array syntax `[]`, null-coalescing `??`, single quotes unless interpolating.
- **Never touch Blesta core assets** — not the compiled `application*.css` / `app.min.js`, and not the core SCSS/JS sources either. Extensions are **self-contained**: ship your own CSS/JS *inside the extension directory* and reference it directly from your views (e.g. `<link>`/`<script src="<?php echo $this->view_dir;?>...">`). Extension CSS/JS does **not** go through the core Gulp build. (The Gulp/SCSS pipeline only applies if you are working on Blesta core itself — which extension authors are not.)
- **Ship `en_us` only** for language in core-style extensions; other locales are added by the translation pipeline. Third-party extensions may ship whatever locales they maintain.
- **Admin vs client UI are different frameworks in 6.0.** **Admin** views render in Paradigm (Bootstrap 5, Bootstrap Icons, vanilla JS — no jQuery) and must work in **both light and dark mode** — see `reference/common/conversion-ui.md`. The **client portal** (`bootstrap` theme) is still **Bootstrap 4 + jQuery + FontAwesome** in 6.0, so client-facing views target BS4 (a BS5 client theme is planned for 6.1). Don't apply the admin conversion to client views.
- **Validate with the `Input` component and set errors** rather than throwing — the host app reads `->errors()` to decide whether to proceed.
- **Log every remote API call** with `$this->log(...)` and mask secrets before logging.

## Status

- ✅ `reference/modules/create.md` — authoring modules (hosting + registrar)
- ✅ `reference/modules/convert.md` — porting modules from 5.x
- ✅ `reference/modules/convert-from-whmcs.md` — porting a WHMCS provisioning/registrar module to Blesta
- ✅ `reference/common/conversion-ui.md` — the Bootstrap 4→5 / jQuery→vanilla / FA→BI UI guide (shared)
- ✅ `reference/gateways/create.md` — authoring gateways (merchant + nonmerchant; classic & tokenized)
- ✅ `reference/gateways/convert.md` — porting gateways from 5.x
- ✅ `reference/gateways/convert-from-whmcs.md` — porting a WHMCS third-party/merchant gateway to Blesta
- ✅ `reference/plugins/create.md` — authoring plugins (pages, actions/widgets, events, cron, ACL, cards, messages, service tabs)
- ✅ `reference/plugins/convert.md` — porting plugins from 5.x
- ✅ `reference/plugins/convert-from-whmcs.md` — porting a WHMCS addon module to a Blesta plugin
