# Porting a WHMCS Payment Gateway to a Blesta 6 Gateway

> **Audience:** Developers (and AI tools) converting a **WHMCS** payment gateway — third-party
> (off-site/redirect) or merchant (card-processing) — into a native **Blesta 6.0** gateway.
>
> **Read `create.md` first** — it is the Blesta-side contract (base `Gateway`, `MerchantGateway` /
> `NonmerchantGateway`, the capability interfaces, the tokenized pattern, the transaction-data return shape).
> This file is the **WHMCS→Blesta mapping**.
>
> **Sources of truth:** the WHMCS SDK samples `sample-gateway-module` (third-party) and
> `sample-merchant-gateway`, including their `modules/gateways/callback/*.php` files, plus
> `developers.whmcs.com/payment-gateways/`. The Blesta side is `components/gateways/lib/` and the shipped
> `paypal_payments_standard` (nonmerchant), `authorize_net` / `stripe_payments` (merchant).
>
> **Shared WHMCS conventions** (licensing caveat, the re-source-don't-transform rule, config field-type
> table, error/logging translation, Smarty→`.pdt`, naming/packaging) live in
> **`reference/common/whmcs-mapping.md`** — read it once; this doc covers only what's gateway-specific.

---

## 0. The mental model

Same paradigm shift as the module port (procedural `+ $params` → OO `+` structured contract), with a
payment-specific twist: **WHMCS splits the off-site flow across a module file *and* a separate callback
script under `modules/gateways/callback/`. Blesta has no callback script — the callback is a *method* on your
gateway class** (`validate()`/`success()`).

| | **WHMCS** | **Blesta** |
|---|---|---|
| Shape | procedural `gatewaymodule_link($params)` | `class GatewayName extends NonmerchantGateway` (or `MerchantGateway`) |
| Two kinds | **Third-party** (`_link` + callback) / **Merchant** (`_capture`/`_3dsecure`) | **Nonmerchant** (`buildProcess`/`validate`/`success`) / **Merchant** (`MerchantCc*` interfaces) |
| Config | `_config()` returns admin fields | `getSettings()` (HTML) + `editSettings()` (validate) + `encryptableFields()` + `setMeta()` |
| Off-site redirect | `_link()` returns an HTML form | `buildProcess()` returns the redirect HTML |
| Callback / IPN | **separate `callback/{name}.php` script** (`addInvoicePayment`) | **`validate()`** (authoritative IPN) + **`success()`** (browser return) methods |
| Card charge | `_capture($params)` (raw card in `$params`) | `processCc($card_info, $amount, …)` (`MerchantCc`) |
| 3-D Secure / SCA | `_3dsecure()` returns a redirect form | `buildPaymentConfirmation()` (`MerchantCcForm`) |
| Result | `return ['status' => 'success'\|'declined'\|'error', 'transid' => …, …]` | return transaction-data `['status' => 'approved'\|…, 'transaction_id' => …, …]` (status enum differs — §7) |
| Logging | `logTransaction(name, data, status)` | `$this->log($url, $data, 'input'\|'output', $success)` |

---

## 1. File & packaging mapping

| WHMCS | Blesta | Notes |
|---|---|---|
| `modules/gateways/{name}.php` (procedural) | `{name}/{name}.php` (class `Name extends NonmerchantGateway` or `MerchantGateway`) | One class |
| `modules/gateways/callback/{name}.php` | **(no file)** — fold into `validate()`/`success()` methods | §5 |
| `_config()` / `_MetaData()` | `config.json` (`name`/`description` lang keys, `icon`, `currencies`, optional `signup_url`) + `getSettings()` | §3–§4 |
| field saved by `_config()` | gateway **meta** via `setMeta()`; secrets listed in `encryptableFields()` | §4 |
| `logo` | `views/default/images/logo.png` | |
| API client (inline in WHMCS) | `apis/` (your HTTP client) | |
| client-facing HTML (`_link`/`_3dsecure`) | `views/default/*.pdt` (`process.pdt`, `cc_form.pdt`, …) | §10 |
| `composer.json` | `composer.json` `type: blesta-gateway-nonmerchant` **or** `blesta-gateway-merchant` | Installs under `components/gateways/{merchant\|nonmerchant}/` |

---

## 2. `$params` → Blesta data shapes

WHMCS gateway functions receive one flat `$params`. In Blesta the equivalent values arrive as **method
arguments** and **gateway meta**.

| WHMCS `$params` key | What it is | Blesta equivalent |
|---|---|---|
| `accountID`, `secretKey`, `testMode`, `dropdownField`, … | your `_config()` fields | gateway **meta** (set in `setMeta()`, read as `$this->meta[...]`); secrets in `encryptableFields()` |
| `invoiceid` | Invoice being paid | the invoice id in `$invoice_amounts` / passed to `buildProcess()` |
| `description` | Invoice description | `options['description']` to `buildProcess()` |
| `amount` | Amount to charge | `$amount` arg (`buildProcess`, `processCc`, `refundCc`) |
| `currency` | Currency code | set via `setCurrency()`; available as `$this->currency` |
| `clientdetails[firstname,lastname,email,address1,address2,city,state,postcode,country,phonenumber]` | Payer profile | `$contact_info` array passed to `buildProcess()` / `$card_info` for merchant (names/address) |
| `cardtype`, `cardnum`, `cardexp` (**MMYY**), `cardstart`, `cardissuenum`, `cccvv` | Raw card data (merchant) | `$card_info` array to `processCc()` — **note the different key names/formats** (next row) |
| ↳ card field renames | — | `cardnum`→`card_number`; `cardexp` (MMYY)→**`card_exp` in `yyyymm`** (reformat!); `cccvv`→**`card_security_code`** (not `cvv`); `cardtype`→`type`; names→`first_name`/`last_name`; address→`address1`/`address2`/`city`/`state`/`zip`/`country` (verified in `components/gateways/lib/merchant_cc.php`) |
| `transid` | Prior transaction id (refund) | `$transaction_id` / `$reference_id` args to `refundCc`/`refund` |
| `subscriptionID` | Gateway subscription id | (no standard Blesta arg — see §5.3) |
| `systemurl` | WHMCS base URL | `Configure::get('Blesta.gw_callback_url')` (nonmerchant) / `Blesta.mgw_callback_url` (merchant) for callbacks (§8) |
| `returnurl` | Post-payment return URL | the return option passed to `buildProcess()` |
| `companyname`, `langpaynow`, `name`, `paymentmethod`, `whmcsVersion` | System/labels | company settings / language / `Configure` as needed |

> **Card data only exists in WHMCS merchant `$params`.** A third-party/nonmerchant gateway never sees the
> PAN — the customer enters it on the gateway's site. Mirror that in Blesta: `NonmerchantGateway` has no
> card methods; only `MerchantGateway` (via `MerchantCc`) receives `$card_info`. **Prefer tokenization**
> (`MerchantCcForm` + `MerchantCcOffsite`) so the raw PAN never touches your code — see `create.md`
> tokenized section.
>
> **The card keys & formats differ — a silent-failure trap.** Blesta's `$card_info` (per
> `components/gateways/lib/merchant_cc.php`) uses **`card_security_code`** (WHMCS sends `cccvv` — a literal
> `cvv` key will be missing/empty and the CVV silently dropped) and **`card_exp` in `yyyymm`** (WHMCS sends
> `cardexp` as **MMYY** — you must reformat, e.g. `'20' . substr($exp,2,2) . substr($exp,0,2)`, or the
> processor gets the wrong expiry). Also `cardnum`→`card_number`, `cardtype`→`type`. `state`/`country` in
> `$card_info` are **arrays** (`['code'=>,'name'=>]` / `['alpha2'=>,'alpha3'=>,'name'=>]`), not the bare
> strings WHMCS passes.

---

## 3. Which Blesta gateway type? (read WHMCS `_MetaData()`)

```
WHMCS _MetaData() flags / which functions exist
  ├─ has _link() + callback/, DisableLocalCreditCardInput=true   → THIRD-PARTY (redirect)
  │     → Blesta NonmerchantGateway  (buildProcess / validate / success)
  └─ has _capture() (+ optional _3dsecure)                        → MERCHANT (card)
        → Blesta MerchantGateway implementing capability interfaces:
            - MerchantCc       (processCc/authorizeCc/captureCc/voidCc/refundCc)
            - MerchantCcForm   (buildCcForm/buildPaymentConfirmation)   ← maps _3dsecure
            - MerchantCcOffsite (storeCc/processStoredCc/…)             ← tokenized, recommended
```

- WHMCS `DisableLocalCreditCardInput => true` + `TokenisedStorage` flags describe a redirect gateway that
  doesn't take local card input → **Nonmerchant** in Blesta.
- A merchant gateway that omits those flags takes card input → **Merchant** in Blesta. Whether you implement
  raw-PAN `MerchantCc` or tokenized `MerchantCcForm`+`MerchantCcOffsite` is your PCI choice (`create.md`).
- `composer.json` `type` must match: `blesta-gateway-nonmerchant` vs `blesta-gateway-merchant`.

---

## 4. `_config()` → settings (both types)

| WHMCS `_config()` element | Blesta |
|---|---|
| `FriendlyName` (`Type => System`) | `config.json` `name` (a language key) — **not** a settings field |
| `accountID` / `secretKey` / … fields | rendered by `getSettings()` (HTML/`InputFields` in `settings.pdt`); validated & returned by `editSettings()`; persisted by `setMeta()` |
| `secretKey` and other secrets | **must be listed in `encryptableFields()`** so they're stored encrypted |
| field `Type` (text/password/yesno/dropdown/radio/textarea) | the corresponding `Form`/field helper in `settings.pdt` (text/password/checkbox/select/radio/textarea) |
| `testMode` (yesno) | a `test_mode` meta toggle read in your processing methods |

The WHMCS config field types (`yesno`→checkbox, `dropdown`→select, `radio`→radio, etc.) map per the shared
table in `reference/common/whmcs-mapping.md` §3 — including the **key-not-label** warning (`_config()`
`fields` is keyed by the config key; `FriendlyName` is display metadata). The **admin settings form is
Paradigm/Bootstrap 5** (§10).

---

## 5. Third-party (nonmerchant) mapping

| WHMCS | Blesta `NonmerchantGateway` | Notes |
|---|---|---|
| `_link($params)` (returns HTML form) | `buildProcess($contact_info, $amount, $invoice_amounts, $options)` | Build the off-site redirect form/button; render via `process.pdt`. Use `Configure::get('Blesta.gw_callback_url')` for the callback URL (§8) |
| `callback/{name}.php` (verify + `addInvoicePayment`) | **`validate($get, $post)`** — the authoritative IPN | Verify the signature/hash here (WHMCS does `md5(...)`); return the **transaction-data array** (`status`, `reference_id`, `transaction_id`, `parent_transaction_id`) **plus** `client_id`, `amount`, `currency`, `invoices` — the host records the payment (no `addInvoicePayment` call) |
| browser-return half of the callback | **`success($get, $post)`** | The user's return from the gateway; usually re-verifies and shows status. `validate()` is authoritative; don't credit twice |
| `_refund($params)` | `refund(...)` (optional on nonmerchant) | Return transaction-data with `status => 'refunded'` (§7) |
| `_cancelSubscription($params)` | *(no standard method)* — §5.3 | |

### 5.1 `_link` → `buildProcess`

WHMCS `_link` hand-builds `<form action="gateway-url">` with hidden fields and a callback URL of
`systemurl . '/modules/gateways/callback/' . $moduleName . '.php'`. In Blesta, `buildProcess()` does the same
job but the callback URL comes from `Configure::get('Blesta.gw_callback_url')` (§8), and you render the form
from `process.pdt` (client portal — **Bootstrap 4**, §10). The customer's billing details arrive in
`$contact_info` rather than `$params['clientdetails']`.

### 5.2 callback script → `validate()` / `success()`

WHMCS's callback file does: `getGatewayVariables` → verify hash → `checkCbInvoiceID` → `checkCbTransID`
(dup check) → `logTransaction` → `addInvoicePayment`. Translate each step:

| WHMCS callback step | Blesta `validate()` |
|---|---|
| `getGatewayVariables($name)` | `$this->meta` (already loaded for your gateway) |
| hash/signature check (`md5(...)`) | **do the same verification**; on mismatch **`$this->Input->setErrors([...])` and return _nothing_ (no array)** — see the warning below |
| `checkCbInvoiceID()` | resolve the invoice from the posted id; return it in the `invoices` array |
| `checkCbTransID()` (duplicate guard) | the host dedupes by `transaction_id`; return a stable `transaction_id` |
| `logTransaction()` | `$this->log($url, serialize($post), 'output', $success)` |
| `addInvoicePayment()` | **don't call anything** — return `status => 'approved'` + `amount`/`currency`/`invoices`; Blesta posts the payment |

> **Returning ANY array records a transaction — even `status => 'error'`.** `GatewayPayments::processNotification()`
> records a transaction for **any array** `validate()` returns; it only **applies it to invoices** when
> `status === 'approved'` (verified in `components/gateway_payments/gateway_payments.php`). So for a
> *genuine processor decline* you can return `status => 'declined'`/`'error'` and it's logged as a
> declined/errored transaction (not applied) — that's correct. But for a **forged/failed hash verification**
> where you want **no record at all**, **set Input errors and return no array** (the `else` branch passes
> your errors up and adds nothing). Don't return `status => 'error'` to "reject" a forged callback — that
> still writes a transaction row.

### 5.3 `_cancelSubscription`

Blesta has **no standard gateway subscription-cancel method** in the nonmerchant contract — Blesta drives
recurring billing itself and re-initiates payment per invoice rather than relying on a gateway-side
subscription. If your gateway genuinely creates provider-side subscriptions, handle cancellation in your own
code/admin action; don't expect a base method. Flag this as a behaviour change during the port.

---

## 6. Merchant mapping

| WHMCS | Blesta | Notes |
|---|---|---|
| `_capture($params)` (raw card; CVV only when customer-present) | `processCc($card_info, $amount, array $invoice_amounts = null)` (`MerchantCc`) | Charge now. `$card_info` uses `card_number`/`card_exp`(yyyymm)/`card_security_code`/`type`/names/address (§2 — **not** `cvv`/MMYY). Return transaction-data (§7) |
| automated recurring `_capture` (no CVV) | `processStoredCc(...)` (`MerchantCcOffsite`) with a stored token | Map customer-present vs recurring to `processCc` vs `processStoredCc`; see `requiresCustomerPresent()` |
| `_3dsecure($params)` (returns redirect form) | `buildPaymentConfirmation(...)` (`MerchantCcForm`) for SCA/3-D Secure | This is the **confirmation UI** step. The async return/webhook is handled by your **`validate()`** reached via `/callback/mgw/` (§8) — **not** `success()`, which is nonmerchant-only |
| (card entry form) | `buildCcForm()` (`MerchantCcForm`) | WHMCS renders the card form itself; in Blesta you provide it via `buildCcForm` (client portal, BS4) |
| `_refund($params)` | `refundCc($reference_id, $transaction_id, $amount)` (`MerchantCc`) / `refundStoredCc(...)` | Return `status => 'refunded'` |
| authorize / capture later | `authorizeCc()` + `captureCc()` (`MerchantCc`) | WHMCS `_capture` is auth+capture in one; split if your processor separates them |
| void | `voidCc()` / `voidStoredCc()` | No direct WHMCS sample equivalent |
| tokenized storage (`MerchantCcOffsite`) | **the full interface**: `storeCc`, `updateCc`, `removeCc`, `processStoredCc`, `authorizeStoredCc`, `captureStoredCc`, `voidStoredCc`, `refundStoredCc`, `requiresCcStorage()` | Recommended. Token arrives in `$card_info['reference_id']`; store + charge off-session (`create.md` tokenized section) |

> **`MerchantCcOffsite` is an all-or-nothing interface — implement every method.** It declares **nine**
> methods (`storeCc`, `updateCc`, `removeCc`, `processStoredCc`, `authorizeStoredCc`, `captureStoredCc`,
> `voidStoredCc`, `refundStoredCc`, `requiresCcStorage()` — verified in
> `components/gateways/lib/merchant_cc_offsite.php`). A class that only implements `processStoredCc`/
> `refundStoredCc` won't satisfy the interface. If your processor lacks an operation, implement the method
> and have it set an error / return a declined-style result rather than omitting it.
>
> **WHMCS `_capture` receives the raw PAN; the modern Blesta pattern avoids that.** If you're porting a
> raw-card merchant gateway, you *can* implement `MerchantCc` (`processCc`) to match WHMCS one-for-one, but
> strongly prefer the tokenized `MerchantCcForm` + `MerchantCcOffsite` route to cut PCI scope — the
> processor's token (e.g. a payment-method id / nonce) replaces the PAN, and recurring charges use
> `processStoredCc()` off-session. `_3dsecure` maps onto `buildPaymentConfirmation()` for SCA.

---

## 7. Return shape & status enum (the key translation)

WHMCS and Blesta use **different status vocabularies** and **different keys**. Translate every `return`:

| WHMCS return key | Blesta transaction-data key |
|---|---|
| `status` | `status` (enum differs — below) |
| `transid` | `transaction_id` |
| (n/a) | `reference_id` (your gateway's reference; often the same value) |
| (n/a) | `parent_transaction_id` (for captures/refunds/voids referencing the original) |
| `rawdata` | **not returned** — log it via `$this->log(...)` instead |
| `declinereason` | `message` |
| `fee` / `fees` | (no transaction-data field — record separately if needed) |

**Status enum mapping:**

| WHMCS `status` | Blesta `status` |
|---|---|
| `success` | `approved` |
| `declined` | `declined` |
| `error` | `error` |
| (refund success) | `refunded` |
| (void) | `void` |
| (pending/async) | `pending` |

Blesta's full enum is `approved | declined | void | pending | reconciled | refunded | returned | error`
(`create.md`). **Failure handling differs by cause:** a *config/validation* failure → `setErrors()` and
abort (no array); a *processor decline/failure* → **return** the array with `status => declined|error` and a
`message`. (Shipped gateways set transaction errors **and** return `status => error` for processor failures.)

---

## 8. Callback / notify URL

Blesta has **two** callback constants — one per gateway type (verified in `config/blesta.php`). Each is a
**URL prefix**; you append the routing segments your gateway needs (commonly `{company_id}/{gateway}/`).

| Type | Constant | Resolves to (prefix) |
|---|---|---|
| **Nonmerchant** | `Configure::get('Blesta.gw_callback_url')` | `…<webdir>/callback/gw/` → append `{company_id}/{gateway}/` |
| **Merchant** | `Configure::get('Blesta.mgw_callback_url')` | `…<webdir>/callback/mgw/` → append your routing segments |

| WHMCS | Blesta |
|---|---|
| `$params['systemurl'] . '/modules/gateways/callback/' . $moduleName . '.php'` | the matching constant above (`gw` for nonmerchant, `mgw` for merchant) + your segments |
| return URL `$params['returnurl']` | the return option you pass into `buildProcess()` |
| nonmerchant IPN | routes to **`validate()`** (`/callback/gw/…`, authoritative) and browser return to **`success()`** |
| merchant `_3dsecure` return → `callback3DSecureRedirect()` | routes to your **`validate()`** via **`/callback/mgw/`** (merchant gateways do **not** have `success()`) |

There is **no per-gateway callback PHP file** in Blesta. Build outbound callback URLs from the correct
constant for your gateway type — `Blesta.gw_callback_url` (nonmerchant) or `Blesta.mgw_callback_url`
(merchant) — not a hand-built `/modules/gateways/callback/...` path. `success()` is a **nonmerchant-only**
method (the browser-return half); merchant confirmation/return is handled by `buildPaymentConfirmation()`
plus `validate()`.

---

## 9. Errors & logging

```php
// WHMCS: return ['status' => 'declined', 'declinereason' => '...', 'rawdata' => $resp];
//        logTransaction($name, $_POST, 'Failure');
//
// Blesta:
$this->log($url, serialize($masked_request),  'input',  true);
$response = $api->charge($request);
$this->log($url, serialize($response), 'output', $approved);

if ($config_problem) {
    $this->Input->setErrors(['gateway' => ['config' => $message]]);   // validation/config → abort, no array
    return;
}
return [
    'status'                => $approved ? 'approved' : 'declined',   // processor result → RETURN the array
    'reference_id'          => $reference,
    'transaction_id'        => $transactionId,
    'parent_transaction_id' => null,
    'message'               => $approved ? '' : $declineReason,
];
```

- `logTransaction()` → paired `$this->log()` (`'input'` request, `'output'` response), **masking the PAN/CVV
  and secrets** first (`maskData()`/`maskDataRecursive()` are available on the base gateway).
- WHMCS "`rawdata` in the return" → Blesta logs it; it is **not** part of the transaction-data array.

---

## 10. Admin vs client UI (6.0)

Same split as everywhere — and gateways have **both** surfaces:

| Surface | WHMCS | Blesta method → view | Theme (6.0) |
|---|---|---|---|
| **Admin settings** | `_config()` form | `getSettings()` → `settings.pdt` | **Paradigm / Bootstrap 5**, Bootstrap Icons, vanilla JS, light+dark (`reference/common/conversion-ui.md`) |
| **Client payment** | `_link()` / `_3dsecure()` / card form | `buildProcess()` / `buildPaymentConfirmation()` / `buildCcForm()` → `process.pdt` / `cc_form.pdt` | **Client portal — Bootstrap 4 + jQuery + FontAwesome** (leave on BS4 for 6.0) |

Convert the **settings** view to Bootstrap 5; keep the **customer-facing payment** forms on Bootstrap 4 for
6.0 (a BS5 client theme arrives in 6.1). The WHMCS `_link`/`_3dsecure` HTML is already plain form markup —
re-author it into `process.pdt` targeting the BS4 client theme.

---

## 11. Conversion checklist

- [ ] Gateway rewritten as a **class** — `NonmerchantGateway` (third-party) or `MerchantGateway` +
      capability interfaces (merchant); `composer.json` `type` matches (`blesta-gateway-nonmerchant` /
      `-merchant`).
- [ ] `_config()` → `getSettings()`/`editSettings()`/`setMeta()`; secrets in `encryptableFields()`.
      `FriendlyName` → `config.json` `name` (lang key). `config.json` declares `currencies` (and
      `signup_url`/`icon`).
- [ ] **No `$params`, no callback script.** Values re-sourced from method args + `$this->meta`; the
      `callback/{name}.php` logic folded into **`validate()`** (authoritative; nonmerchant **and** merchant
      webhooks) and, for **nonmerchant only**, **`success()`** (browser return).
- [ ] Nonmerchant: `_link` → `buildProcess` (callback URL from **`Blesta.gw_callback_url`** → `/callback/gw/`);
      callback verify + `addInvoicePayment` → `validate()` returning transaction-data + `client_id`/`amount`/
      `currency`/`invoices` (host records payment). Forged/failed verification → `setErrors()` + **no array**.
- [ ] Merchant: `_capture` → `processCc` (`$card_info` uses `card_security_code` + `card_exp` **yyyymm**, not
      `cvv`/MMYY) or `processStoredCc` for recurring; `_3dsecure` → `buildPaymentConfirmation`; merchant
      webhooks → `validate()` via **`Blesta.mgw_callback_url`** → `/callback/mgw/` (**no `success()`**);
      `_refund` → `refundCc`/`refundStoredCc`. **Prefer tokenized** `MerchantCcForm`+**full** `MerchantCcOffsite`
      (all 9 methods) over raw-PAN `MerchantCc`.
- [ ] Return shape translated: `transid`→`transaction_id`, add `reference_id`/`parent_transaction_id`,
      `declinereason`→`message`, `rawdata`→logged (not returned); **status enum** mapped
      (`success`→`approved`, etc.).
- [ ] Failure handling: config/validation → `setErrors()` + abort (no array); processor decline/error →
      **return** `status => declined|error` with a `message` (this **does** record a transaction — only a
      forged/invalid callback should `setErrors()` + return nothing).
- [ ] `logTransaction` → paired `$this->log()` input/output with PAN/CVV/secrets masked.
- [ ] `_cancelSubscription` behaviour noted (no standard Blesta method; Blesta drives recurring itself).
- [ ] **Admin** `settings.pdt` converted to Bootstrap 5/Paradigm; **client** payment views left on BS4 +
      jQuery + FontAwesome.

---

## 12. See also

- `create.md` — the full Blesta gateway contract (base + merchant/nonmerchant + capability interfaces + the
  tokenized PCI-reducing pattern). **All mapping targets reference it.**
- `convert.md` — the 5.x→6.0 gateway port (UI/cosmetics) — complementary if you're also modernizing the UI.
- `reference/common/conversion-ui.md` — admin (Paradigm/BS5) view conversion for `settings.pdt`.
- `reference/modules/convert-from-whmcs.md` — the module-side WHMCS port; shares the `$params`/config-field/
  logging/error patterns.
- WHMCS samples (`sample-gateway-module`, `sample-merchant-gateway`) + `developers.whmcs.com/payment-gateways/`
  — the WHMCS-side source of truth.
