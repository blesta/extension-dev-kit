# Blesta 6 Extension Development Kit

A reference kit for **building** and **converting** Blesta 6.0 extensions — modules, plugins, and payment
gateways — optimized for AI-assisted development.

## Two ways to use it

1. **Portable context (any AI tool).** The files under `reference/` are plain markdown. Drop the relevant
   one into Cursor / Copilot / ChatGPT / Claude.ai / the API as context and ask it to build or convert an
   extension. Load the file for the type you're working on, plus:
   - any **WHMCS** port also needs `reference/common/whmcs-mapping.md` (the shared conventions);
   - any **admin UI / view** work also needs `reference/common/conversion-ui.md`.

   Otherwise the per-type files stand alone.

2. **Claude Code skill.** This folder is also a skill: `SKILL.md` is the router that pulls in the right
   reference on demand. To install, copy or symlink this folder into your Claude Code skills directory
   (e.g. `~/.claude/skills/extension-dev-kit`), then invoke it from Claude Code.

## Layout

```
extension-dev-kit/
├── SKILL.md                         # skill router (also a human index)
├── README.md                        # this file
└── reference/
    ├── common/
    │   ├── conversion-ui.md         # Bootstrap 4→5 / jQuery→vanilla / FA→BI / Paradigm UI (shared)
    │   └── whmcs-mapping.md         # shared WHMCS→Blesta conventions (license, re-source rule, field types, errors/logging, Smarty→pdt)
    ├── modules/
    │   ├── create.md                # author a new module (hosting + registrar)
    │   ├── convert.md               # port a module from 5.x
    │   └── convert-from-whmcs.md    # port a WHMCS provisioning/registrar module to Blesta
    ├── gateways/
    │   ├── create.md                # author a new gateway (merchant + nonmerchant; classic & tokenized)
    │   ├── convert.md               # port a gateway from 5.x
    │   └── convert-from-whmcs.md    # port a WHMCS third-party/merchant gateway to Blesta
    └── plugins/
        ├── create.md                # author a new plugin (pages, actions/widgets, events, cron, ACL, cards, messages, tabs)
        ├── convert.md               # port a plugin from 5.x
        └── convert-from-whmcs.md    # port a WHMCS addon module to a Blesta plugin
```

## Scope today

- ✅ **Modules** — full method-surface reference for hosting/server (`extends Module`) and registrar
  (`extends RegistrarModule`) modules, plus a 5.x→6.0 porting guide and a **WHMCS→Blesta** porting guide
  (provisioning + registrar).
- ✅ **Payment gateways** — base `Gateway` + `MerchantGateway`/`NonmerchantGateway` contracts, all
  capability interfaces, the modern tokenized (PCI-reducing) pattern, a 5.x→6.0 porting guide, and a
  **WHMCS→Blesta** porting guide (third-party + merchant).
- ✅ **Plugins** — the full handler contract plus directly-routable controllers/models, navigation &
  widget actions, the event system, cron tasks, ACL permissions, dashboard/profile cards, messenger
  templates, and service tabs, with a 5.x→6.0 porting guide and a **WHMCS addon→Blesta plugin** porting guide.
- ✅ **UI conversion** — the shared Bootstrap 5 / Paradigm guide used by every extension type.

## Sourcing & accuracy

The references are distilled from the Blesta 6.0.0-b2 abstract base classes and shipped extensions:

- **Modules** — `components/modules/module.php`, `registrar_module.php`; examples `cpanel` (hosting) and
  `namesilo` (registrar).
- **Gateways** — `components/gateways/lib/gateway.php`, `merchant_gateway.php`, `nonmerchant_gateway.php`,
  and the capability interfaces; examples `authorize_net` (classic merchant), `stripe_payments` (tokenized
  merchant), `paypal_payments_standard` (nonmerchant).
- **Plugins** — `components/plugins/lib/plugin.php` (the `Plugin` base class), `app/models/plugin_manager.php`
  (the install/upgrade/registration engine), the `Actions`/`Navigation`/`CronTasks`/`Permissions` models,
  `core/Util/Events/` (the event system), `config/routes.php`, and `app/controllers/admin_company_plugins.php`
  (the "Manage" dispatcher); examples `support_manager` (full-featured), `order`, `feed_reader` (minimal),
  `client_cards`, `webhooks`.

When in doubt, the base classes are the source of truth — see the "Canonical reference files" table at the
end of each `create.md`.
