# Porting a Blesta Payment Gateway from 5.x to 6.0

> **Audience:** Developers (and AI tools) converting an existing 5.x gateway to run natively in Blesta 6.0.
>
> Like modules, conversion is **two jobs**: (1) the **admin settings UI** (Bootstrap 4→5, jQuery→vanilla,
> FA→Bootstrap Icons) and (2) a **small PHP pass**. The processing contract itself is unchanged.

---

## 1. Will the old gateway even run?

Yes — the `Gateway` / `MerchantGateway` / `NonmerchantGateway` contracts are backwards compatible. The
method signatures (`processCc`, `validate`, `success`, `setMeta`, `editSettings`, `encryptableFields`, …)
and the transaction-data return shape (`status`/`reference_id`/`transaction_id`/`parent_transaction_id`/
`message`) are the same in 6.0. What needs attention is the **UI** and a few cosmetics.

---

## 2. Admin vs client — only the admin side converts to BS5

| Method | Surface | Host theme (6.0) | Action |
|---|---|---|---|
| `getSettings()` → `settings.pdt` | **Admin** (Payment Gateways) | Paradigm | **Convert to Bootstrap 5** (see `reference/common/conversion-ui.md`) |
| `buildProcess`, `buildAuthorize`, `buildCcForm`, `buildAchForm`, `buildPaymentConfirmation`, `buildAchVerificationForm` → `process.pdt` etc. | **Client** (checkout / pay page) | `bootstrap` portal — still BS4 + jQuery + FontAwesome | **Leave on BS4** — converting breaks them |

A gateway's biggest (often only) admin view is the **settings form**. Convert that; leave customer-facing
payment forms as Bootstrap 4 for 6.0. (A BS5 client theme is planned for 6.1.)

---

## 3. The admin UI conversion

Apply `reference/common/conversion-ui.md` to `getSettings()` output / `settings.pdt`:

- Form controls: pass `'class' => 'form-control'` / `'form-select'` / `'form-check-input'` on every
  `Form` helper call; `form-group` → `mb-3`; `<ul><li>` field rows → `<div class="mb-3">`.
- Buttons/badges/utilities: `btn-default` → `btn-secondary`, `pull-right` → `float-end`, etc.
- `data-toggle` → `data-bs-toggle` (and the rest of the `-bs-` infix family).
- Icons in the settings view: `fas fa-…` → `bi bi-…`.
- Any jQuery that shows/hides settings fields (common — e.g. toggling test-mode or API-mode fields):
  rewrite to vanilla JS (`DOMContentLoaded`, `querySelectorAll`, `addEventListener`). The admin theme has
  no jQuery.
- Verify in light **and** dark mode.

---

## 4. The PHP pass (small)

- **config.json `icon`** — add `"icon": "bi-…"` (name only — the admin prepends the `bi` base class;
  `"bi bi-…"` is wrong).
- **Icon strings in admin PHP arrays** (if any) → `bi bi-…`.
- **No signature/return changes.** Don't touch `processCc`/`validate`/`success`/`editSettings`/
  `encryptableFields` shapes.
- **Consider tokenization** while you're here: if your gateway passes raw card data through `processCc`,
  evaluate moving to the offsite/tokenized interfaces (`MerchantCcForm` + `MerchantCcOffsite`) to cut PCI
  scope — see `create.md` §8.3 and the "Modern tokenized merchant gateway" section. This is an enhancement,
  not required for a 6.0 compatibility port.

---

## 5. What does *not* change

- Base contract: `setCurrency`, `getSettings`, `editSettings`, `encryptableFields`, `setMeta`.
- Merchant capability interfaces and their method names.
- Nonmerchant `buildProcess` / `validate` / `success` and the callback (`Blesta.gw_callback_url`) flow.
- The transaction-data return shape and `status` enum.
- `$this->log(...)`, `maskData(...)`, `getCommonError(...)`, `currencies` in config.json.
- `apis/`, `language/`, `views/default/` layout.

---

## 6. Conversion checklist

- [ ] **Admin** `settings.pdt` / `getSettings()` output converted to BS5 (per `conversion-ui.md`).
- [ ] **Client** payment forms (`process.pdt`, `buildProcess`/`buildCcForm` output) left on BS4 + jQuery + FontAwesome.
- [ ] jQuery in the **settings** view rewritten to vanilla; client form JS left as-is.
- [ ] `config.json` `"icon"` added as name-only (`bi-credit-card`); admin icon strings → `bi bi-…`; client icons unchanged.
- [ ] Settings view verified in light **and** dark mode.
- [ ] No processing-method signature/return changes.
- [ ] *(Optional enhancement)* evaluate tokenized/offsite interfaces to reduce PCI scope.

---

## 7. See also

- `create.md` — full gateway contract (base + merchant/nonmerchant + capability interfaces) and the modern tokenized pattern.
- `reference/common/conversion-ui.md` — the admin UI conversion guide (shared).
