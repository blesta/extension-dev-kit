# Authoring a Blesta 6 Payment Gateway

> **Audience:** Developers (and AI tools) building a **new** Blesta 6.0 payment gateway from scratch.
> For porting a 5.x gateway, read `convert.md` first, then use this as the contract reference.
>
> **Source of truth:** the base classes `components/gateways/lib/gateway.php` (`Gateway`),
> `merchant_gateway.php` (`MerchantGateway`), `nonmerchant_gateway.php` (`NonmerchantGateway`), the
> capability interfaces in the same `lib/` dir, plus the shipped `authorize_net` (merchant) and
> `paypal_payments_standard` (nonmerchant) gateways.

---

## 0. Mental model

A **gateway** moves money. Blesta owns invoices, transactions, and payment accounts; the gateway owns
*"charge this / refund this / tell me when an off-site payment happened."* There are two fundamentally
different kinds:

| | **Merchant** gateway | **Nonmerchant** gateway |
|---|---|---|
| Base class | `MerchantGateway` | `NonmerchantGateway` |
| Who collects card data | **Blesta** (or the processor via a token) — server-to-server API | The **gateway's own site** — customer leaves Blesta and pays there |
| Customer flow | Customer enters card on the Blesta pay page; you call the processor API | Customer is redirected/POSTed to the gateway, pays, comes back |
| Core methods | `processCc` / `processAch` (+ authorize/capture/void/refund, optional offsite storage) | `buildProcess` (render the redirect form) + `validate` (IPN) + `success` (return) |
| Examples | authorize_net, stripe_payments, braintree | paypal_payments_standard, stripe (checkout), coinbase_commerce, offline |
| composer `type` | `blesta-gateway-merchant` | `blesta-gateway-nonmerchant` |

Both extend a shared abstract `Gateway` that handles metadata, settings, encryption, logging, and currency.

**The transaction-data array is the universal return shape** (memorise it — §3): every processing method
returns it.

---

## 1. Choose the type

```
Does the customer enter their card/payment details ON YOUR/BLESTA's page,
and do you call a remote API server-to-server to charge it?
  ├─ YES → MERCHANT gateway (extends MerchantGateway, implements MerchantCc/MerchantAch/…)
  └─ NO (customer is sent to the processor's site to pay) → NONMERCHANT gateway (extends NonmerchantGateway)
```

A modern merchant gateway should still avoid handling raw PANs by using the **offsite/tokenized**
interfaces (`MerchantCcOffsite`) — see §8.3. "Merchant" here means *Blesta orchestrates the charge*, not
that you must touch card numbers.

---

## 2. Directory layout

```
your_gateway/
├── your_gateway.php          # main class: class YourGateway extends MerchantGateway|NonmerchantGateway
├── config.json              # metadata (see §3)
├── composer.json            # "type": "blesta-gateway-merchant" | "blesta-gateway-nonmerchant"
├── README.md
├── apis/                     # remote API client wrapper(s) — your code
│   └── your_gateway_api.php
├── language/
│   └── en_us/
│       └── your_gateway.php  # $lang['YourGateway.*']
└── views/
    └── default/
        ├── images/logo.png
        ├── settings.pdt      # ADMIN settings form (rendered by getSettings) — Paradigm/BS5
        └── process.pdt       # CLIENT payment form (nonmerchant buildProcess / merchant buildCcForm) — BS4
```

File/class naming follows the camel/snake convention: `your_gateway.php` → class `YourGateway`.

---

## 3. `config.json` and the universal return shape

### 3.1 config.json

Loaded in the constructor via `loadConfig()` into `$this->config`. Gateways have a **different** key set
from modules — there is **no `type` or `module` block**; instead they declare `currencies` and an optional
`signup_url`:

```json
{
    "version": "1.0.0",
    "name": "YourGateway.name",
    "description": "YourGateway.description",
    "authors": [
        { "name": "Your Company", "url": "https://example.com" }
    ],
    "currencies": ["USD", "EUR", "GBP"],
    "signup_url": "https://example.com/signup",
    "icon": "bi-credit-card"
}
```

| Key | Meaning |
|---|---|
| `version` | Drives `upgrade($current_version)` |
| `name` / `description` | **Language keys** (run through `Language::_()` by `getName`/`getDescription`) |
| `authors[]` | name + url |
| `currencies` | ISO 4217 codes this gateway supports — Blesta only offers the gateway for these (`getCurrencies()`) |
| `signup_url` | Optional link shown in admin to sign up for the processor (`getSignupUrl()`) |
| `icon` | Bootstrap Icon **name only**, e.g. `bi-credit-card` — the admin renders `<i class="bi {icon}">`, so do **not** write `bi bi-…` |
| `logo` | Optional path; defaults to `views/default/images/logo.png` |

### 3.2 The transaction-data return shape (every processing method)

`processCc`, `processAch`, `processStoredCc`, `capture*`, `void*`, `refund*`, and the nonmerchant
`validate`/`success`/`capture`/`void`/`refund` all return an array of this shape:

```php
return [
    'status'                => 'approved',   // see enum below — REQUIRED
    'reference_id'          => 'xxxx',        // gateway-only reference (optional)
    'transaction_id'        => 'abc123',      // the processor's id for THIS transaction
    'parent_transaction_id' => 'abc000',      // original txn id (refunds/captures of a prior auth)
    'message'               => 'Approved',    // extra UI message beyond the standard status text (optional)
];
```

**`status` enum (exact strings):** `approved`, `declined`, `void`, `pending`, `reconciled`, `refunded`,
`returned`, **`error`**. Map the processor's response codes to these.

> The core base-class docblocks list only the first seven and omit `error`, but the shipped gateways
> **return `error` heavily** for failed/unknown processor outcomes (authorize_net, stripe_payments, and
> paypal_payments_standard all do). Treat `error` as a valid status for a processor failure.

`validate()` / `success()` (nonmerchant) additionally return `client_id`, `amount`, `currency`, and
`invoices` (an array of `['id' => invoice_id, 'amount' => x]`).

**Failures — two cases, don't conflate them:**

- **Validation / configuration failures** (missing settings, malformed input, gateway can't be reached
  at all): set Input errors (`$this->Input->setErrors($this->getCommonError(...))`, §7/§8.6) and abort —
  don't return a transaction array.
- **Processor responses** (the processor was reached and declined/failed): return the normal
  transaction-data array with the appropriate status — `declined`, `pending`, or `error` — *and* you may
  also set transaction errors for the UI. authorize_net, for example, sets transaction errors **and**
  returns `['status' => 'error', …]`.

---

## 4. Class skeleton & constructor

```php
<?php
class YourGateway extends MerchantGateway implements MerchantCc   // or: extends NonmerchantGateway
{
    private $meta;       // gateway settings, set via setMeta()
    private $currency;   // set via setCurrency()

    public function __construct()
    {
        $this->loadConfig(dirname(__FILE__) . DS . 'config.json');
        Loader::loadComponents($this, ['Input']);
        Language::loadLang('your_gateway', null, dirname(__FILE__) . DS . 'language' . DS);
    }
}
```

`composer.json` must set `"type": "blesta-gateway-merchant"` or `"blesta-gateway-nonmerchant"` so the
Blesta composer installer drops it in `components/gateways/{merchant|nonmerchant}/`.

---

## 5. The base `Gateway` contract (settings lifecycle)

Every gateway implements these (the five abstract ones are mandatory):

```php
abstract public function setCurrency($currency);            // store ISO 4217 code for subsequent calls
abstract public function getSettings(array $meta = null);   // return HTML settings form (admin)
abstract public function editSettings(array $meta);         // validate + return meta to store (or refill form on error)
abstract public function encryptableFields();               // array of meta keys to encrypt at rest
abstract public function setMeta(array $meta = null);       // receive the stored settings for this instance
```

Plus inherited concrete helpers you usually leave alone: `getName`, `getDescription`, `getVersion`,
`getAuthors`, `getCurrencies`, `getLogo`, `getIcon`, `getSignupUrl`, `errors()`, and lifecycle no-ops
`install()`, `upgrade($current_version)`, `uninstall($gateway_id, $last_instance)`.

**Settings flow:** `setMeta()` hands you the saved settings (decrypted) before any processing call —
stash them in `$this->meta`. `getSettings()` renders the admin form; `editSettings()` validates submitted
settings and returns the array to persist; `encryptableFields()` lists which of those keys are stored
encrypted (API keys, secrets, passwords).

```php
public function setMeta(array $meta = null)        { $this->meta = $meta; }
public function setCurrency($currency)             { $this->currency = $currency; }
public function encryptableFields()                { return ['api_key', 'secret']; }

public function getSettings(array $meta = null)
{
    $this->view = $this->makeView('settings', 'default',
        str_replace(ROOTWEBDIR, '', dirname(__FILE__) . DS));
    Loader::loadHelpers($this, ['Form', 'Html']);
    $this->view->set('meta', $meta);
    return $this->view->fetch();
}

public function editSettings(array $meta)
{
    $rules = [
        'api_key' => [
            'valid' => [
                'rule' => 'isEmpty', 'negate' => true,
                'message' => Language::_('YourGateway.!error.api_key.valid', true)
            ]
        ],
    ];
    $this->Input->setRules($rules);
    $this->Input->validates($meta);
    return $meta;   // returned even on error so the form repopulates; caller checks errors()
}
```

---

## 6. Logging & masking (inherited)

```php
protected function log($url, $data = null, $direction = 'input', $success = false)  // returns 8-char group id
protected function maskData(array $data, array $mask_fields, $mask_char = 'x', $unmask_length = 0)
protected function maskDataRecursive(array $data, array $mask_fields, ...)
```

**Log every request and response**, masking secrets first. The masking helpers are purpose-built for this
(e.g. leave last 4 of a card visible): `$this->log($url, serialize($this->maskData($req, ['card_number' =>
['char' => 'x', 'length' => -4]])), 'input', $success)`.

---

## 7. Errors

```php
$this->Input->setErrors($this->getCommonError('general'));   // or a custom error array
if ($this->Input->errors()) { /* abort */ }
```

`getCommonError($type)` differs by base class:

- **MerchantGateway:** `card_number_invalid`, `card_expired`, `routing_number_invalid`,
  `account_number_invalid`, `duplicate_transaction`, `card_not_accepted`, `invalid_security_code`,
  `address_verification_failed`, `transaction_not_found`, `unsupported`, `general` (each maps to the right
  form field).
- **NonmerchantGateway:** `invalid`, `transaction_not_found`, `unsupported`, `general`.

---

## 8. Merchant gateways

```php
class YourGateway extends MerchantGateway implements MerchantCc, MerchantCcOffsite { … }
```

`MerchantGateway` adds one abstract method and an HTTP helper:

```php
abstract public function requiresCustomerPresent();   // true if a human must be present (e.g. CVV) — blocks autodebit
protected function httpRequest($method, $url = null, $params = null);   // wraps the Net Http component
```

You then **implement capability interfaces à la carte** — only the ones your processor supports.

### 8.1 Capability interfaces (pick what you support)

| Interface | Methods | Use when |
|---|---|---|
| `MerchantCc` | `processCc`, `authorizeCc`, `captureCc`, `voidCc`, `refundCc` | You charge raw card data via API |
| `MerchantCcForm` | `buildCcForm`, `buildPaymentConfirmation` | You render a custom client-side card form (e.g. processor JS/iframe) |
| `MerchantCcOffsite` | `storeCc`, `updateCc`, `removeCc`, `processStoredCc`, `authorizeStoredCc`, `captureStoredCc`, `voidStoredCc`, `refundStoredCc`, `requiresCcStorage` | Cards are vaulted at the processor (tokenized) — **preferred for PCI** |
| `MerchantAch` | `processAch`, `voidAch`, `refundAch` | You debit bank accounts |
| `MerchantAchForm` | `buildAchForm` | Custom client-side ACH form |
| `MerchantAchOffsite` | `storeAch`, `updateAch`, `removeAch`, `processStoredAch`, `voidStoredAch`, `refundStoredAch`, `requiresAchStorage` | Bank accounts vaulted at the processor |
| `MerchantAchVerification` | `buildAchVerificationForm`, `verifyAch` | Micro-deposit / bank verification |

`requiresCcStorage()` / `requiresAchStorage()` are called **after** `setMeta()` — return `true` to tell
Blesta to use the offsite (stored) methods for this configured instance, `false` to use the direct ones.
This lets one gateway support both modes based on its settings.

### 8.2 Direct card processing (`MerchantCc`)

`$card_info` arrives with `first_name`, `last_name`, `card_number`, `card_exp` (yyyymm),
`card_security_code`, `type`, address fields, `state` (array `code`/`name`), `country` (array
`alpha2`/`alpha3`/`name`), `zip`. `$amount` is the charge; `$invoice_amounts` is `[['id'=>, 'amount'=>], …]`.

**Example — `authorize_net::processCc()` (the canonical shape):**

```php
public function processCc(array $card_info, $amount, array $invoice_amounts = null)
{
    $this->loadApi('AIM');
    $response = $this->AuthorizeNetAim->authCaptureCc($card_info, $amount, $invoice_amounts);
    $this->logAim($response);              // module-private logging wrapper around $this->log()

    $this->setTransactionErrors($response);

    $status = 'error';
    switch ($response['x_response_code'] ?? null) {
        case '1': $status = 'approved'; break;
        case '2': $status = 'declined'; break;
        case '3': $status = 'error';    break;
        case '4': $status = 'pending';  break;
    }

    return [
        'status'         => $status,
        'reference_id'   => substr($card_info['card_number'] ?? '', -4),
        'transaction_id' => $response['x_trans_id'] ?? null,
        'message'        => $response['x_response_reason_text'] ?? null,
    ];
}
```

`authorizeCc` is the same but auth-only; `captureCc($reference_id, $transaction_id, $amount, …)` settles a
prior auth; `voidCc`/`refundCc` reference the original `transaction_id`.

### 8.3 Offsite / tokenized processing (`MerchantCcOffsite`) — preferred

Store the payment method at the processor and charge the token later (Blesta never stores the PAN). **Exact
signatures matter here** — the stored capture/void/refund methods take *more* parameters than their direct
`captureCc`/`voidCc`/`refundCc` counterparts (note the extra `$transaction_reference_id`):

```php
public function storeCc(array $card_info, array $contact, $client_reference_id = null);
public function updateCc(array $card_info, array $contact, $client_reference_id, $account_reference_id);
public function removeCc($client_reference_id, $account_reference_id);
public function processStoredCc($client_reference_id, $account_reference_id, $amount, array $invoice_amounts = null);
public function authorizeStoredCc($client_reference_id, $account_reference_id, $amount, array $invoice_amounts = null);
public function captureStoredCc($client_reference_id, $account_reference_id, $transaction_reference_id, $transaction_id, $amount, array $invoice_amounts = null);
public function voidStoredCc($client_reference_id, $account_reference_id, $transaction_reference_id, $transaction_id);
public function refundStoredCc($client_reference_id, $account_reference_id, $transaction_reference_id, $transaction_id, $amount);
public function requiresCcStorage();   // return true to make Blesta use the stored path
```

- `storeCc(...)` returns `['client_reference_id'=>, 'reference_id'=>, 'expiration'=>(yyyymm), 'last4'=>,
  'type'=>]` (or `false` on failure). All `process/authorize/capture/void/refundStoredCc` return the
  transaction-data array (§3.2).

**`MerchantAchOffsite` is similar but NOT identical** — it has `storeAch`, `updateAch`, `removeAch`,
`processStoredAch`, `voidStoredAch`, `refundStoredAch`, `requiresAchStorage`, but **no `authorizeStoredAch`
/ `captureStoredAch`** (ACH has no separate auth/capture step). Exact signatures:

```php
public function storeAch(array $account_info, array $contact, $client_reference_id = null);
public function updateAch(array $account_info, array $contact, $client_reference_id, $account_reference_id);
public function removeAch($client_reference_id, $account_reference_id);
public function processStoredAch($client_reference_id, $account_reference_id, $amount, array $invoice_amounts = null);
public function voidStoredAch($client_reference_id, $account_reference_id, $transaction_reference_id, $transaction_id);
public function refundStoredAch($client_reference_id, $account_reference_id, $transaction_reference_id, $transaction_id, $amount);
public function requiresAchStorage();
```

(Likewise the direct `MerchantAch` interface is only `processAch` / `voidAch` / `refundAch` — no
`authorizeAch`/`captureAch`.)

### 8.4 Client-side payment forms (`MerchantCcForm` / `MerchantAchForm`)

`buildCcForm()` returns HTML for the card form shown to the **customer** (e.g. a processor JS/iframe so the
PAN never hits your server); `buildPaymentConfirmation($reference_id, $transaction_id, $amount)` renders a
post-payment confirmation. **These render in the client portal — see §10 for the BS4/BS5 distinction.**

### 8.5 Modern tokenized merchant gateway (the recommended pattern)

Most **new** merchant gateways should be **tokenized**, not local-storage: the card (PAN/CVC) never touches
Blesta, which dramatically reduces the merchant's PCI scope. This is how `stripe_payments` works, and how
`authorize_net_acceptjs` works (Accept.js nonce) — by contrast, classic `authorize_net` stores the full
card in Blesta and posts it server-to-server (higher PCI scope).

**The recipe:** implement **`MerchantCcForm` + `MerchantCcOffsite`** (and the ACH equivalents if
supported), and return `true` from `requiresCcStorage()` so Blesta always uses the stored/token path.

```php
class StripePayments extends MerchantGateway implements
    MerchantCc, MerchantCcForm, MerchantCcOffsite,
    MerchantAch, MerchantAchForm, MerchantAchOffsite, MerchantAchVerification
{
    public function requiresCcStorage() { return true; }   // always tokenize; never store the PAN in Blesta
}
```

**The token flow (PAN-free):**

1. **`buildCcForm()`** renders the processor's *client-side* tokenizer instead of raw `<input>`s — e.g.
   Stripe Elements (a `#card-element`) or Authorize.Net Accept.js. For Stripe it first creates a server-side
   `SetupIntent` and passes its `client_secret` to the view.
2. In the browser, the processor JS exchanges the card for a **token** (Stripe PaymentMethod `pm_xxx`,
   Accept.js nonce, …). Only that token is put into a hidden field and submitted — the PAN/CVC are POSTed
   directly to the processor, never to Blesta.
3. The token arrives in your PHP as **`$card_info['reference_id']`**. This is the key convention: in a
   tokenized gateway, `card_info` carries a token in `reference_id`, *not* a card number.
4. **`storeCc($card_info, $contact, $client_reference_id = null)`** turns the token into a durable vault
   record — attach it to a processor customer, and return the reference ids Blesta will persist:

   ```php
   return [
       'client_reference_id' => $customer->id,     // e.g. Stripe cus_xxx — the customer/vault id
       'reference_id'        => $card_info['reference_id'],  // e.g. pm_xxx — the stored method id
       'last4'               => $card->card->last4,
       'expiration'          => $card->card->exp_year . $card->card->exp_month,  // yyyymm
       'type'                => $this->mapCardType($card->card->brand),
   ];
   ```

5. **`processStoredCc($client_reference_id, $account_reference_id, $amount, $invoice_amounts = null)`**
   charges the vaulted method off-session (no CVC) and returns the standard transaction-data array:

   ```php
   $payment = Stripe\PaymentIntent::create([
       'amount'         => $this->formatAmount($amount, $this->currency),
       'currency'       => $this->currency,
       'customer'       => $client_reference_id,   // cus_xxx
       'payment_method' => $account_reference_id,  // pm_xxx
       'confirm'        => true,
       'off_session'    => true,
   ]);
   return [
       'status'         => $status,                  // approved | declined | pending | …
       'transaction_id' => $payment['latest_charge'],
       'message'        => $payment['error']['message'] ?? null,
   ];
   ```

6. **`processCc()`** (if you implement it at all) **delegates** to the stored flow using the token in
   `$card_info['reference_id']` — it must **not** expect a raw card number:

   ```php
   public function processCc(array $card_info, $amount, array $invoice_amounts = null)
   {
       return $this->processStoredCc(null, $card_info['reference_id'], $amount, $invoice_amounts);
   }
   ```

**SCA / 3-D Secure.** Tokenized card flows often need a customer challenge. When the processor returns a
"requires action" state, surface it through **`buildPaymentConfirmation($reference_id, $transaction_id,
$amount)`**, which renders a client-side confirmation (Stripe: `stripe.handleCardPayment(client_secret)`).
Asynchronous results (settlement, refunds, disputes) are reconciled via the processor's **webhooks** —
`stripe_payments` ships a webhook handler (a `validate()`-style method) for `charge` / `payment_intent`
events that looks the transaction up and maps the processor status to the Blesta enum. Confirm the webhook
entry point/route against your target build rather than assuming one.

**Bundling the SDK.** A tokenized gateway typically vendors the processor's PHP SDK under its own directory
(e.g. `vendor/stripe/stripe-php`) and loads it explicitly:
`Loader::load(dirname(__FILE__) . DS . 'vendor' . DS . 'stripe' . DS . 'stripe-php' . DS . 'init.php');`
then `Stripe\Stripe::setApiKey($this->meta['secret_key']);`.

**Pick your reference by model:**

| Gateway | Model | Implements | PAN in Blesta? |
|---|---|---|---|
| `stripe_payments` | Tokenized (Elements + PaymentIntents) | `MerchantCcForm` + `MerchantCcOffsite` (+ ACH) | No (`pm_xxx`/`cus_xxx` only) |
| `authorize_net_acceptjs` | Tokenized (Accept.js nonce) | `MerchantCcForm` + `MerchantCcOffsite` | No (nonce only) |
| `authorize_net` | Classic local storage | `MerchantCc` (+ CIM offsite) | Yes (raw card, higher PCI scope) |

---

## 9. Nonmerchant gateways

```php
class YourGateway extends NonmerchantGateway { … }
```

Three methods carry the flow:

```php
public function buildProcess(array $contact_info, $amount, array $invoice_amounts = null, array $options = null);
abstract public function validate(array $get, array $post);   // IPN/webhook — authoritative
abstract public function success(array $get, array $post);    // customer returned from the gateway
```

Optional: `buildAuthorize(...)` (auth-only form), and `capture`/`void`/`refund` (default to "unsupported").

### 9.1 `buildProcess` — send the customer to the processor

Return HTML (usually an auto-submitting `<form>` of hidden fields, or a redirect) that POSTs the
customer to the processor. Encode the invoice/amount data so it round-trips back via the callback, and set
the **notify URL** (IPN) and **return URL**.

**Example — paypal_payments_standard (abridged):**

```php
$fields = [
    'cmd'           => '_xclick',
    'business'      => $this->meta['account_id'],          // merchant email
    'amount'        => round($amount, 2),
    'currency_code' => $this->currency,
    'item_name'     => $options['description'],
    'custom'        => $this->serializeInvoices($invoice_amounts),  // 'id1=amt1|id2=amt2'
    'notify_url'    => Configure::get('Blesta.gw_callback_url')
                        . Configure::get('Blesta.company_id') . '/your_gateway/?client_id=' . $client_id,
    'return'        => $options['return_url'],
    'rm'            => '2',                                 // POST back on return
    // + contact_info fields (first_name, last_name, address1, city, country, zip, state)
];
return $this->buildForm($post_to_url, $fields);            // your helper: HTML form + submit
```

`$options` may include a `recur` block (`amount`, `term`, `period`) for subscriptions, plus `description`
and `return_url`. `$contact_info` is the full billing contact (see the base-class docblock for fields).

### 9.2 The callback URL & how `validate`/`success` get called

Build the notify URL from `Configure::get('Blesta.gw_callback_url')` + company id + your gateway name.
Blesta exposes a gateway callback endpoint; when the processor POSTs the IPN there, Blesta dispatches it to
your gateway's **`validate($get, $post)`**. When the customer clicks "return to merchant," Blesta calls
**`success($get, $post)`**.

> Confirm the exact callback constant/route against your target build (`Blesta.gw_callback_url` is what the
> shipped gateways use); don't hard-code a path.

### 9.3 `validate` (IPN) vs `success` (return)

- **`validate`** is **authoritative** — verify authenticity (e.g. PayPal: POST the payload back with
  `cmd=_notify-validate` and require `VERIFIED`, and check `receiver_email` matches your configured
  account), guard against duplicate processing, map the processor status to the Blesta enum, and return the
  full transaction-data array (incl. `client_id`, `amount`, `currency`, `invoices`). Set
  `getCommonError('invalid')` if verification fails.
- **`success`** handles the browser return; it's typically lighter (decode invoices, return
  `status => 'approved'`) because `validate` is the trusted path. Never grant service on `success` alone if
  the IPN hasn't confirmed.

**Return shape (validate/success):**

```php
return [
    'client_id'             => $client_id,
    'amount'                => $post['mc_gross'],
    'currency'              => $post['mc_currency'],
    'status'                => $status,            // approved|pending|refunded|declined|…
    'transaction_id'        => $post['txn_id'],
    'parent_transaction_id' => $post['parent_txn_id'] ?? null,
    'invoices'              => $this->unserializeInvoices($post['custom']),  // [['id'=>,'amount'=>], …]
];
```

`clientIdFromEmail($email)` (inherited) helps resolve the client when the processor only returns an email.

---

## 10. Where gateway UI renders — admin vs client (6.0)

Same split as modules — **do not** convert client payment forms to Bootstrap 5 for 6.0:

| Method | Surface | Host theme (6.0) | Write in… |
|---|---|---|---|
| `getSettings()` | **Admin** (Payment Gateways settings) | **Paradigm** | Bootstrap 5, vanilla JS, Bootstrap Icons, light+dark — see `reference/common/conversion-ui.md` |
| `buildProcess()`, `buildAuthorize()`, `buildCcForm()`, `buildAchForm()`, `buildPaymentConfirmation()`, `buildAchVerificationForm()` | **Client** (checkout / pay page) | **`bootstrap` client portal** — Bootstrap 4 + jQuery + FontAwesome | Bootstrap 4, jQuery OK, FontAwesome |

So `settings.pdt` is BS5; `process.pdt` (the customer payment form) is BS4 for 6.0. (A BS5 client theme is
planned for 6.1.)

---

## 11. Minimal skeletons

**Nonmerchant:**

```php
<?php
class YourGateway extends NonmerchantGateway
{
    private $meta;
    public function __construct()
    {
        $this->loadConfig(dirname(__FILE__) . DS . 'config.json');
        Loader::loadComponents($this, ['Input']);
        Language::loadLang('your_gateway', null, dirname(__FILE__) . DS . 'language' . DS);
    }
    public function setCurrency($currency) { $this->currency = $currency; }
    public function setMeta(array $meta = null) { $this->meta = $meta; }
    public function encryptableFields() { return ['secret_key']; }
    public function getSettings(array $meta = null) { /* return settings.pdt HTML */ }
    public function editSettings(array $meta) { /* validate */ return $meta; }

    public function buildProcess(array $contact_info, $amount, array $invoice_amounts = null, array $options = null)
    { /* return auto-submit form HTML to the processor */ }

    public function validate(array $get, array $post)
    { /* verify IPN; return transaction-data array */ }

    public function success(array $get, array $post)
    { /* handle browser return; return transaction-data array */ }
}
```

**Merchant (direct CC):**

```php
<?php
class YourGateway extends MerchantGateway implements MerchantCc
{
    private $meta;
    public function __construct() { /* loadConfig + Input + lang as above */ }
    public function setCurrency($currency) { $this->currency = $currency; }
    public function setMeta(array $meta = null) { $this->meta = $meta; }
    public function encryptableFields() { return ['api_login', 'api_key']; }
    public function requiresCustomerPresent() { return true; }
    public function getSettings(array $meta = null) { /* settings.pdt */ }
    public function editSettings(array $meta) { /* validate */ return $meta; }

    public function processCc(array $card_info, $amount, array $invoice_amounts = null) { /* charge; return txn array */ }
    public function authorizeCc(array $card_info, $amount, array $invoice_amounts = null) { /* auth only */ }
    public function captureCc($reference_id, $transaction_id, $amount, array $invoice_amounts = null) { /* settle */ }
    public function voidCc($reference_id, $transaction_id) { /* void */ }
    public function refundCc($reference_id, $transaction_id, $amount) { /* refund */ }
}
```

```json
// config.json
{
    "version": "1.0.0",
    "name": "YourGateway.name",
    "description": "YourGateway.description",
    "authors": [{ "name": "You", "url": "https://example.com" }],
    "currencies": ["USD"],
    "icon": "bi-credit-card"
}
```

---

## 12. Author's checklist

- [ ] Correct base class (`MerchantGateway` vs `NonmerchantGateway`) and composer `type`.
- [ ] `config.json` `name`/`description` are language keys that resolve in `language/en_us/`; `currencies` set; `icon` is name-only (`bi-credit-card`).
- [ ] Five base abstracts implemented: `setCurrency`, `getSettings`, `editSettings`, `encryptableFields`, `setMeta`.
- [ ] Secrets listed in `encryptableFields()` and masked before `$this->log(...)`.
- [ ] Every processing method returns the transaction-data array with a valid `status` enum value.
- [ ] Merchant: only the capability interfaces you actually support are implemented; `requiresCcStorage()`/`requiresAchStorage()` reflect the configured mode; `requiresCustomerPresent()` correct for autodebit.
- [ ] **New merchant gateways: prefer the tokenized pattern** (§8.5) — `MerchantCcForm` + `MerchantCcOffsite`, `requiresCcStorage()` = `true`, token arrives in `$card_info['reference_id']`, PAN never reaches Blesta. Handle SCA via `buildPaymentConfirmation()` + webhooks.
- [ ] Nonmerchant: `validate()` independently verifies authenticity and guards duplicates; `success()` doesn't grant service the IPN hasn't confirmed; invoices round-trip via the callback payload.
- [ ] **Admin** `settings.pdt` is Paradigm/BS5; **client** payment forms are BS4 + jQuery + FontAwesome for 6.0 (§10).
- [ ] Callback/notify URL built from `Blesta.gw_callback_url` (not hard-coded).

---

## 13. Canonical reference files

| What | Path (in a 6.0 build) |
|---|---|
| `Gateway` base | `components/gateways/lib/gateway.php` |
| `MerchantGateway` base | `components/gateways/lib/merchant_gateway.php` |
| `NonmerchantGateway` base | `components/gateways/lib/nonmerchant_gateway.php` |
| Capability interfaces | `components/gateways/lib/merchant_{cc,ach}{,_offsite,_form}.php`, `merchant_ach_verification.php` |
| Merchant example (CC + ACH + offsite) | `components/gateways/merchant/authorize_net/` |
| Modern tokenized example | `components/gateways/merchant/stripe_payments/` |
| Nonmerchant example (redirect + IPN) | `components/gateways/nonmerchant/paypal_payments_standard/` |
| UI conversion rules (admin) | `reference/common/conversion-ui.md` |
