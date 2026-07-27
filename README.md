# invoq REST API

**English** · [Bahasa Indonesia](./docs/README.id.md) · [Español](./docs/README.es-419.md) · [Français](./docs/README.fr.md) · [Português](./docs/README.pt-BR.md) · [Tiếng Việt](./docs/README.vi.md) · [Türkçe](./docs/README.tr.md) · [ไทย](./docs/README.th.md) · [简体中文](./docs/README.zh-Hans.md) · [繁體中文](./docs/README.zh-Hant.md)

Stablecoin payments for indie developers. Non-custodial — funds settle straight to your wallet.

This is the reference for invoq's public REST API. The official SDKs — [Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby) — wrap exactly these endpoints.

- **Base URL:** `https://api.invoq.money`
- **Hosted checkout:** `https://pay.invoq.money/<invoice id>`
- **Dashboard** (API keys, receiving wallet, webhooks): `https://app.invoq.money`

**Coding with AI? Paste this.**

```
Add stablecoin payments to my project with invoq. Start in test mode. Read the docs before you write any code: https://invoq.money/llms.txt
```

## How it works

1. **Create an invoice** from your server (`POST /v1/invoices`).
2. **Let the buyer pay it.** Easiest: send them to `https://pay.invoq.money/<invoice id>` — it shows the amount, address, and QR code, speaks ten languages, and needs zero UI work from you. Or embed the same checkout with [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). The buyer sends USDT or USDC from any wallet or exchange.
3. **Get told when it's paid.** invoq confirms the transfer on-chain and posts an `invoice.paid` webhook to your server. Settlement goes straight to your own wallet.

## Quickstart

Grab a test key (`sk_test_...`) from the dashboard, then create your first invoice:

```bash
curl https://api.invoq.money/v1/invoices \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "12.34",
    "description": "Website audit for June",
    "reference_id": "order_10086"
  }'
```

For a live invoice, the `id` in the response is all you need — `https://pay.invoq.money/<id>` is your checkout page. This one is a test invoice, which carries no on-chain payment instructions, so simulate the payment instead:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

If you've set a test webhook URL in the dashboard, it receives a signed `invoice.paid`. That's the whole loop — the rest of this document is the detail.

## Authentication

Server endpoints take a secret API key created in the dashboard:

```http
Authorization: Bearer sk_test_...
```

- `sk_test_...` keys create **test invoices**: payments are simulated, nothing moves on-chain, and webhooks are real — signed and delivered like live ones.
- `sk_live_...` keys create **live invoices** with real on-chain payment instructions.

Mode always comes from the key; the request body never accepts a `mode` field. Keep secret keys on your server, never in client code.

`GET /v1/invoices/{id}` needs no key. Invoice ids are unguessable public ids used in payment links, so checkout UIs can poll it straight from the browser — CORS allows any origin for `GET` and `HEAD`.

## Requests and responses

Successful responses wrap the resource in `data`:

```json
{ "data": { "id": "inv_..." } }
```

Errors share one shape across every endpoint:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- Branch on `code`. `message` is for humans and can change without notice.
- `fields` appears only for field-level validation errors; `location` is `body`, `query`, or `path`.
- Extra business context lands in `meta` — `retry_after`, `reason_codes`, `min_amount`, and so on.

Conventions worth knowing:

- **Validation is strict.** An unrecognized body key or query parameter returns `400 invalid_request` with `fields[].code: "unknown_field"` — appending a cache-buster fails the request.
- **Amounts are decimal strings, never floats.** Invoice amounts carry 4 fractional digits, paid and due amounts 18, and token amounts inside `payment_options` exactly `token_decimals`.
- **`429 rate_limited` puts its hint in the body**, as `meta.retry_after` in whole seconds. No `Retry-After` header is sent — read `meta`.
- Request bodies are limited to 4KB; anything larger is `413 request_body_too_large`.
- Every JSON response carries `Cache-Control: no-store` — payment state is polled, so nothing may serve you a stale invoice.
- `GET /` is an unauthenticated liveness probe. It returns `204 No Content`.

Invoice creation is rate limited per project: live 3,000/minute and 100,000/day, test 300/minute and 10,000/day.

## Create an invoice

### `POST /v1/invoices`

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Field | Notes |
| --- | --- |
| `amount` | Required. Decimal string, `0.01`–`1000000.00`, up to 2 fractional digits. Normalized in responses (`12.34` → `12.3400`). Currency is fixed to `USD` and returned in responses; it is not a request field. |
| `reference_id` | Optional reference of your own, unique per project and mode, max 200 chars. Retrying with identical terms returns the existing invoice with `200 OK` and `meta.result: "reused"`; different terms return `409 reference_id_conflict`. |
| `description` | Optional payer-visible text, max 500 chars. |
| `return_url` | Optional `http(s)` URL, max 1000 chars — the merchant return button shown after payment. Omit it and the project's default is snapshotted; pass `null` or `""` for no return URL. On a `reference_id` retry an omitted `return_url` is not checked against the existing invoice, so pass it explicitly when the retry must assert a value. |

`201 Created`, or `200 OK` on idempotent reuse:

```json
{
  "data": {
    "id": "inv_0123456789abcdefghjk",
    "mode": "live",
    "amount": "12.3400",
    "currency": "USD",
    "reference_id": "order_10086",
    "description": "Website audit for June",
    "return_url": "https://example.com/orders/order_10086",
    "status": "unpaid",
    "checkout_status": "open",
    "payment_revision": 0,
    "amount_due": "12.340000000000000000",
    "amount_overpaid": "0.000000000000000000",
    "monitoring_ends_at": "2026-07-26T10:00:00.000Z",
    "payment_options": [
      {
        "collection_method": "direct_exact",
        "chain_namespace": "tron",
        "chain_reference": "0x2b6653dc",
        "currency": "USD",
        "token_address": "TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t",
        "token_decimals": 6,
        "network_label": "TRON",
        "display_symbol": "USDT",
        "logo_url": null,
        "chain_logo_url": null,
        "status": "ready",
        "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
        "invoice_amount": "12.340000",
        "matching_increment": "0.009999",
        "exact_amount": "12.349999"
      },
      {
        "collection_method": "evm_deposit",
        "chain_namespace": "eip155",
        "chain_reference": "8453",
        "currency": "USD",
        "token_address": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
        "token_decimals": 6,
        "network_label": "Base",
        "display_symbol": "USDC",
        "logo_url": null,
        "chain_logo_url": null,
        "status": "ready",
        "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca",
        "suggested_amount": "12.340000"
      }
    ]
  }
}
```

- **Funds can't be redirected through this API.** The request cannot set recipient addresses or contract configuration — those come from your verified project settings. After creation an invoice is immutable except for its payment and settlement state.
- Test invoices return `monitoring_ends_at: null`, `payment_options: []`, and `checkout_status: "unavailable"`. Pay them through the [test-payments endpoint](#post-v1invoicesidtest-payments).

Error codes: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (field code `invalid_format`, or `amount_too_small` / `amount_too_large` with `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

`409 no_payment_options_available` means no payment option could be issued, and carries sorted `meta.reason_codes`: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted`. `merchant_address_provisioning` is transient — a new Solana or TRON address is still activating, and the same request usually succeeds seconds later.

## Read an invoice

### `GET /v1/invoices/{id}`

The payer-facing view of an invoice: summary, payment state, project branding, payment instructions, and the receipt trail. No API key required — this is the endpoint checkout UIs poll.

```json
{
  "data": {
    "id": "inv_0123456789abcdefghjk",
    "mode": "live",
    "amount": "12.3400",
    "currency": "USD",
    "description": "Website audit for June",
    "return_url": "https://example.com/orders/order_10086",
    "project": { "id": "proj_...", "name": "Acme store", "logo_url": null },
    "status": "settled",
    "checkout_status": "paid",
    "payment_revision": 1,
    "amount_paid": "12.340000000000000000",
    "amount_due": "0.000000000000000000",
    "amount_overpaid": "0.000000000000000000",
    "transfers": [
      {
        "chain_namespace": "solana",
        "chain_reference": "5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp",
        "transaction_id": "2Ana1pUpv2ZbMVkwF5FXapYeBEjdxDatLn7nvJkhgTSXbs59SyZSx866bXirPgj8QQVB57uxHJBG1YFvkRbFj4T",
        "event_index": 2,
        "amount": "12.340000000000000000",
        "explorer_transaction_url": "https://solscan.io/tx/2Ana1pUpv2Zb..."
      }
    ],
    "monitoring_ends_at": "2026-07-26T10:00:00.000Z",
    "payment_options": [ { "...": "..." } ]
  }
}
```

Compared with the create response this adds `amount_paid`, `project`, and `transfers`, and omits your `reference_id`. `description`, `return_url`, `project.name`, and `project.logo_url` are all nullable — a checkout has to render without any of them.

`transfers` is the receipt trail, so a checkout can show each on-chain transaction and link to an explorer. Each entry carries `chain_namespace`, `chain_reference`, `transaction_id`, `event_index`, `amount` (invoice currency at the same 18-digit scale as `amount_paid`; for `direct_exact` it excludes the matching increment), and `explorer_transaction_url` (`null` when none is configured).

Only confirmed transfers appear — a pending one could still be dropped by a reorg — capped at the 20 largest by amount, largest first, so dust can't crowd out the real payment. Always present: `[]` until a transfer confirms, and always `[]` for test invoices.

Invalidly shaped and unknown ids both return `404 invoice_not_found`.

## How the buyer pays

`payment_options` lists the ways this invoice can be paid, one entry per network and token, in payer-facing order: USDT before USDC, then each token's network order. The options, and the addresses and amounts in them, are fixed when the invoice is created — a receiving address or rail you configure later doesn't rewrite an existing invoice. Only each option's `status` is re-evaluated on every response.

Identify an option by (`chain_namespace`, `chain_reference`, `token_address`) — never by array position, and never by `network_label`, `display_symbol`, `logo_url`, or `chain_logo_url`, which are display metadata.

`collection_method` is the discriminator.

**`evm_deposit`** — a deposit address belonging to this invoice alone:

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

Any positive, on-time transfer to it credits the invoice by its amount. `suggested_amount` is guidance, not a match requirement: it's `max(0, amount_due − pending)` rounded **up** to the rail's decimals, so it can exceed `amount_due` by up to one token unit. Don't assert the two are equal.

**`direct_exact`** — your Solana or TRON address plus an exact amount:

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

The buyer must send exactly `exact_amount` (`invoice_amount + matching_increment`) in one transfer. The increment is how the payment is attributed to this invoice; it reaches you, but it is never invoice credit.

Since that exact amount always covers the whole invoice, a `direct_exact` option turns `unavailable` the moment the invoice has any confirmed or pending payment, including one that arrived on a different rail. For the same reason a partly paid direct invoice can't be topped up — issue a new invoice for the balance.

Only a `status: "ready"` option carries the payable fields above. An `unavailable` one carries the common fields alone — it's out of service (manual review, a blocked address or rail, a paused chain, an elapsed payment window) and shouldn't be offered to the buyer.

## Payment status

Two status fields, answering different questions.

**`status`** is the canonical accounting status, backed by confirmed payments and settlement: `unpaid`, `partially_paid`, `paid`, `settling`, `settled`, or `review_required`. `paid`, `settling`, and `settled` all mean the buyer paid — they differ only in how far the funds have progressed to your wallet. `review_required` means the invoice is pending manual review — it is **not** a paid state, so don't fulfill on it even when `amount_paid` looks sufficient.

**`checkout_status`** is the payer-facing state, derived on every response and evaluated in this order:

| Value | Meaning |
| --- | --- |
| `paid` | `status` is `paid`, `settling`, or `settled` |
| `confirming` | On-chain evidence has arrived and is not yet confirmed |
| `expired` | Past `monitoring_ends_at` |
| `open` | At least one payment option is `ready` |
| `unavailable` | Everything else — review, blocked routes, unpaid test invoices |

**`checkout_status` never authorizes fulfillment.** Use the `invoice.paid` webhook.

**`payment_revision`** starts at `0` and increments once whenever the confirmed credited payment set changes: a new transfer, a reversal, a new test payment, or a correction to a credited transfer's chain time. Settlement alone doesn't move it, and it can change while `status` doesn't. Use it to discard an invoice snapshot or webhook that arrived after a newer one.

`amount_due` is `max(amount − amount_paid, 0)` and `amount_overpaid` is `max(amount_paid − amount, 0)`. Read those fields instead of subtracting money yourself.

## The payment window

`monitoring_ends_at` is one day after the invoice was created, and it is the only boundary. A transfer is credited automatically only when its own on-chain time falls inside the window — nothing from before the invoice existed, nothing at or after `monitoring_ends_at`. Your clock, our observation time, and webhook arrival time play no part.

A late payment isn't lost. It's recorded against the invoice and shown in the dashboard, where you can credit it by naming the transaction — a judgment call nothing automatic will make on your behalf. How long you have depends on the rail:

- **EVM** — no deadline. The deposit address belongs to this invoice alone and is never reissued, so a transfer that reaches it can't have paid anything else.
- **Solana and TRON** — 21 days from creation. The exact amount stays reserved for 20 days past `monitoring_ends_at`; after that it can belong to a newer invoice, and nobody can say which one a late transfer paid. What counts is when the transfer landed, not when you get to it.

One consequence for your integration: **`invoice.paid` can arrive long after checkout showed `expired`**, and no field distinguishes it. If you cancel or re-sell an order when its checkout expires, reconcile `invoice.paid` against your own order state rather than assuming the invoice is still open — and process it idempotently either way.

## Webhooks

Configure webhook URLs in the dashboard. Test and live each have their own URL and signing secret.

### Events

**`invoice.paid`** — the invoice reached a paid state (`paid`, `settling`, or `settled`):

```json
{
  "id": "wdel_...",
  "type": "invoice.paid",
  "mode": "live",
  "created_at": "2026-07-19T10:04:00.000Z",
  "data": {
    "invoice": {
      "id": "inv_...",
      "mode": "live",
      "status": "settled",
      "amount": "12.3400",
      "currency": "USD",
      "amount_paid": "12.340000000000000000",
      "reference_id": "order_10086",
      "payment_revision": 1,
      "fully_paid_at": "2026-07-19T10:03:58.000Z"
    }
  }
}
```

**`invoice.payment_reversed`** — a previously paid invoice dropped back below its amount, for example because a chain reorg removed a credited transfer. Same payload shape, with the invoice's current `status`, a higher `payment_revision`, and `fully_paid_at: null`. Treat it as invalidating the earlier fulfillment signal, according to your own business policy.

- `review_required` never triggers `invoice.paid`. If the threshold is crossed during review, the event is sent once, after review clears.
- A true paid → reversed → paid sequence delivers `invoice.paid`, `invoice.payment_reversed`, then a new `invoice.paid`, each with its resulting `payment_revision`.
- `reference_id` and `fully_paid_at` are nullable but always present. `return_url` and payment instructions are deliberately absent — reconcile server-side by invoice id plus `reference_id`.

### Verifying signatures

Every delivery carries:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` is a Unix timestamp in seconds. `v1` is the lowercase hex HMAC-SHA256 of `<t>.<raw_body>`, keyed with that mode's webhook secret. Verify against the **raw request body** before parsing — re-serializing the JSON can change the bytes and invalidate the signature. Reject timestamps outside your replay-tolerance window. The official SDKs ship a `verifyWebhook` helper.

### Delivery and retries

- Delivery POSTs time out after 10 seconds.
- Every non-2xx response — **including redirects and 4xx** — plus network errors and timeouts is retried with bounded backoff: 1 minute, 5 minutes, 30 minutes, then 2 hours, each with up to 20% jitter, for five attempts in total.
- Delivery is **at-least-once and may be out of order.** Deduplicate by event `id`, keep the snapshot with the highest `payment_revision`, and respond `2xx` quickly — do the work after acknowledging.

## Test mode and going live

### `POST /v1/invoices/{id}/test-payments`

Adds a simulated payment to a **test** invoice and returns its updated payment state. Available with `sk_test_...` keys only — this is how you drive the full unpaid → paid → webhook loop without touching a chain.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` is required and greater than zero, up to 15 integer and 4 fractional digits (`5`, `5.0`, and `5.0000` all normalize to `5.0000`).
- `reference_id` is optional, max 200 chars, and idempotent per invoice: the same reference with the same normalized amount returns `200 OK` and `meta.result: "reused"`, a different amount returns `409 test_payment_reference_conflict`.
- Partial, full, and over payments are all allowed: `partially_paid` while `0 < amount_paid < amount`, `paid` once `amount_paid >= amount`. The first crossing into `paid` sends one `invoice.paid`, and every new payment increments `payment_revision`.
- Limited to 300 per minute and 10,000 per day per project.

The response is the invoice in the create shape, plus `amount_paid` and `fully_paid_at`, with `meta.result`.

Error codes: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

### Going live

Once the loop works against your test webhook — a tunnel like ngrok or cloudflared is fine for local dev:

1. Create an `sk_live_` key in the dashboard.
2. Set your live webhook URL.
3. Switch the key in your server config.

Nothing else changes: same endpoints, same shapes, and live invoices now carry real `payment_options`. Test invoices and test payments never touch a chain and are never counted as real payments.

## Support

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
