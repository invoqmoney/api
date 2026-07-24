# invoq REST API

**English** · [Bahasa Indonesia](./docs/README.id.md) · [Español](./docs/README.es-419.md) · [Français](./docs/README.fr.md) · [Português](./docs/README.pt-BR.md) · [Tiếng Việt](./docs/README.vi.md) · [Türkçe](./docs/README.tr.md) · [ไทย](./docs/README.th.md) · [简体中文](./docs/README.zh-Hans.md) · [繁體中文](./docs/README.zh-Hant.md)

Stablecoin payments for indie developers. Non-custodial — funds settle straight to your wallet.

This is the reference for invoq's public REST API. If you use one of the official SDKs ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)), they wrap exactly these endpoints — this document is the contract they follow.

- **Base URL:** `https://api.invoq.money`
- **Hosted checkout:** `https://pay.invoq.money/<invoice id>`
- **Dashboard** (API keys, receiving wallet, webhooks): `https://app.invoq.money`

## How it works

1. **Create an invoice** from your server (`POST /v1/invoices`).
2. **Let the buyer pay it.** Easiest: send them to the hosted checkout at `https://pay.invoq.money/<invoice id>` — it shows the amount, address, and QR code, speaks ten languages, and needs zero UI work from you. Or embed the same checkout in your own site with [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). The buyer sends USDT or USDC from any wallet or exchange.
3. **Get told when it's paid.** invoq confirms the transfer on-chain and sends an `invoice.paid` webhook to your server; settlement goes straight to your own wallet.

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

For a live invoice, the `id` from the response is all you need — `https://pay.invoq.money/<id>` is your checkout page. This one is a test invoice, which carries no on-chain payment instructions, so simulate the payment instead:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

If you've set a test webhook URL in the dashboard, it receives a signed `invoice.paid`. That's the whole loop — the rest of this document is the detail.

## Authentication

Server endpoints use a secret API key created in the dashboard:

```http
Authorization: Bearer sk_test_...
```

- `sk_test_...` keys create **test invoices**: payments are simulated, nothing moves on-chain, and webhooks are real (signed and delivered like live ones).
- `sk_live_...` keys create **live invoices** with real on-chain payment instructions.

The invoice mode always comes from the key — the request body never accepts a `mode` field. Keep secret keys on your server; never ship them in client code.

## Response envelope

Successful responses wrap the resource in `data`:

```json
{ "data": { "id": "inv_..." } }
```

Error responses share one shape across all endpoints:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` is a stable machine-readable error code — branch on this, not on `message`.
- `fields` is present only for field-level validation errors.
- Extra business context is returned in `meta`, such as `retry_after`, `reason_codes`, or `min_amount`.

Request bodies are limited to 4KB; oversized bodies return `413 request_body_too_large`.

## Create an invoice

### `POST /v1/invoices`

Creates an invoice and returns its summary and payment instructions.

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
| `amount` | Required. Decimal string, `0.01`–`1000000.00`, up to 2 fractional digits. Normalized in responses (`12.34` → `12.3400`). The business currency is fixed to `USD` and returned in responses; it is not a request field. |
| `reference_id` | Optional caller-side reference, unique per project + mode, max 200 chars. Retrying with identical terms returns the existing invoice with `200 OK`; different terms return `409 reference_id_conflict`. |
| `description` | Optional payer-visible text, max 500 chars. |
| `return_url` | Optional `http(s)` URL shown as the merchant return button after payment, max 1000 chars. Omitted → the project's default return URL is snapshotted. Explicit `null` or `""` → no return URL. On `reference_id` retries an omitted `return_url` is not validated against the existing invoice — pass it explicitly when the retry must assert a specific value. |

Successful response (`201 Created`; idempotent reuse returns `200 OK` with `meta.result: "reused"`):

```json
{
  "data": {
    "id": "inv_...",
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
    "monitoring_ends_at": "2026-06-14T12:34:56.000Z",
    "payment_options": [
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
      },
      {
        "collection_method": "direct_exact",
        "chain_namespace": "solana",
        "chain_reference": "5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp",
        "currency": "USD",
        "token_address": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "token_decimals": 6,
        "network_label": "Solana",
        "display_symbol": "USDC",
        "logo_url": null,
        "chain_logo_url": null,
        "status": "ready",
        "recipient_address": "GmaDrppBC7P5ARKV8g3djiwP89vz1jLK23V2GBjuAEGB",
        "invoice_amount": "12.340000",
        "matching_increment": "0.000123",
        "exact_amount": "12.340123"
      }
    ]
  }
}
```

Semantics worth knowing:

- **Funds can't be redirected through this API.** The request cannot set recipient addresses or contract configuration — those come from your verified project settings when the invoice is created. After creation an invoice is immutable except for its payment and settlement state.
- **`payment_options` lists the ways the buyer can pay**, one entry per network + token, in payer-facing order (USDT before USDC, then each token's reviewed network order). `collection_method` is the discriminator:
  - `evm_deposit` — a per-invoice EVM deposit address. Any positive on-time transfer credits by its amount; `suggested_amount` (`max(0, amount_due − pending)`) is guidance, not a match requirement.
  - `direct_exact` — a Solana/TRON merchant address with an exact amount. The buyer must send exactly `exact_amount` (`invoice_amount + matching_increment`) in one transfer; the increment is how the payment is attributed and is never invoice credit.
  - Only a `status: "ready"` option carries the payable fields above; an `unavailable` option carries only the common fields. Identify an option by (`chain_namespace`, `chain_reference`, `token_address`), never by array position or display metadata.
- **`checkout_status` is the payer-facing state**, derived on every response: `paid` (canonical status is paid/settling/settled), `confirming` (pending on-chain evidence), `expired` (past `monitoring_ends_at`), `open` (at least one ready option), or `unavailable`. It never authorizes fulfillment — use the `invoice.paid` webhook.
- **`payment_revision`** starts at `0` and increments exactly once whenever the confirmed credited payment set changes (each new test payment too). Use it to discard an older invoice snapshot or webhook delivered after a newer one.
- **Amounts are decimal strings, never floats.** Paid/due amounts use 18 fractional digits. `amount_due` is `max(amount − amount_paid, 0)` and `amount_overpaid` is `max(amount_paid − amount, 0)` — read those fields instead of subtracting money yourself.
- **invoq watches the chain for 7 days** after creation (`monitoring_ends_at`). A transfer landing at or after that instant is recorded but credits nothing; the dashboard's manual reconcile is the operator backstop for edge cases inside the window.
- Test invoices return `monitoring_ends_at: null`, `payment_options: []`, and `checkout_status: "unavailable"` — payments are simulated via the test-payments endpoint below.
- Rate limits per project: live 3,000/minute and 100,000/day; test 300/minute and 10,000/day.

Error codes: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (with `amount_too_small` / `amount_too_large` field codes and `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (with sorted `meta.reason_codes`: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — `merchant_address_provisioning` is transient and usually clears within a minute or two), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Read an invoice

### `GET /v1/invoices/{id}`

Returns the public invoice summary, payer-visible payment state, project branding, and payment instructions. **No API key required** — invoice ids are shareable, unguessable public ids used in payment-link URLs, so this is the endpoint payment UIs poll (CORS allows any origin for GET).

```json
{
  "data": {
    "id": "inv_...",
    "mode": "live",
    "amount": "12.3400",
    "currency": "USD",
    "description": "Website audit for June",
    "return_url": "https://example.com/orders/order_10086",
    "project": { "id": "proj_...", "name": "Acme store", "logo_url": "https://..." },
    "status": "unpaid",
    "checkout_status": "confirming",
    "payment_revision": 0,
    "amount_paid": "0.000000000000000000",
    "amount_due": "12.340000000000000000",
    "amount_overpaid": "0.000000000000000000",
    "transfers": [],
    "monitoring_ends_at": "2026-06-14T12:34:56.000Z",
    "payment_options": [ { "...": "..." } ]
  }
}
```

- `status` is the canonical invoice status backed by confirmed payment and settlement events: `unpaid`, `partially_paid`, `paid`, `settling`, `settled`, or `review_required`.
- `review_required` means the invoice is pending manual review. It is **not** a paid state — do not fulfill on it, even if `amount_paid` looks sufficient.
- `checkout_status` is the derived payer-facing state described above; live invoices with pending on-chain evidence show `confirming` while canonical `status` stays unchanged until the transfer confirms.
- `transfers` is the payer-facing receipt trail: confirmed inbound transfers that credited this invoice, so a checkout can show each on-chain transaction and link a block explorer. Each entry carries `chain_namespace`, `chain_reference`, canonical `transaction_id`, `event_index`, `amount` (invoice-currency units at the same 18-fractional-digit scale as `amount_paid`; for `direct_exact` it excludes the matching increment), and `explorer_transaction_url` (or `null`). Only confirmed transfers appear — a pending one could still be dropped by a reorg — capped at the 20 largest by amount so dust sent to a public deposit address can't crowd out the real payment. Always present: `[]` until a transfer confirms, and always `[]` for test invoices.
- The caller-only `reference_id` is omitted here; only payer-facing `project` branding fields are returned.
- Invalidly shaped and unknown ids both return `404 invoice_not_found`.

## Simulate a test payment

### `POST /v1/invoices/{id}/test-payments`

Adds a simulated payment to a **test** invoice and returns the updated payment state. Only available with `sk_test_...` keys — this is how you exercise the full unpaid → paid → webhook loop without touching a chain.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` is required, must be greater than zero, up to 15 integer and 4 fractional digits (`5`, `5.0`, `5.0000` normalize to `5.0000`).
- `reference_id` is optional, max 200 chars, and idempotent per invoice: reusing it with the same normalized amount returns `200 OK` with `meta.result: "reused"`; a different amount returns `409 test_payment_reference_conflict`.
- Partial, full, and over payments are allowed: `partially_paid` while `0 < amount_paid < amount`, `paid` once `amount_paid >= amount`. The first transition into `paid` triggers one logical `invoice.paid` webhook, and each created payment increments `payment_revision`.
- Creation is limited to 300 per minute and 10,000 per day per project.

Error codes: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhooks

Configure webhook URLs in the dashboard — test and live each have their own URL and signing secret.

### Events

**`invoice.paid`** — sent when an invoice transitions into a paid state (`paid`, `settling`, or `settled`):

```json
{
  "id": "wdel_...",
  "type": "invoice.paid",
  "mode": "test",
  "created_at": "2026-06-10T10:00:00.000Z",
  "data": {
    "invoice": {
      "id": "inv_...",
      "mode": "test",
      "status": "paid",
      "amount": "12.3400",
      "currency": "USD",
      "amount_paid": "13.000000000000000000",
      "reference_id": "order_10086",
      "payment_revision": 1,
      "fully_paid_at": "2026-06-10T10:00:00.000Z"
    }
  }
}
```

**`invoice.payment_reversed`** — sent when a previously-paid invoice drops back below its amount (for example a chain reorg removed a credited transfer). Same payload shape, with the invoice's current `status`, `amount_paid`, a higher `payment_revision`, and `fully_paid_at: null`. Treat it as invalidating the earlier fulfillment signal, according to your own business policy.

- `review_required` never triggers `invoice.paid`. Only after review clears into a paid state is the event created.
- A true paid → reversed → paid sequence delivers `invoice.paid`, `invoice.payment_reversed`, then a new `invoice.paid`, each with its resulting `payment_revision`.
- `reference_id` and `fully_paid_at` are nullable but always present; `return_url` and payment instructions are intentionally absent. Reconcile server-side by invoice id plus `reference_id`.

### Verifying signatures

Every outbound webhook includes:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` is a Unix timestamp in seconds. `v1` is the lowercase hex HMAC-SHA256 of `<t>.<raw_body>`, keyed with the mode-specific webhook secret. Verify against the **raw request body** before parsing — re-serializing the JSON can change the bytes and invalidate the signature. Reject timestamps outside your replay-tolerance window. (The official SDKs ship a `verifyWebhook` helper.)

### Delivery and retries

- Delivery POSTs time out after 10 seconds.
- Every non-2xx response — **including redirects and 4xx** — plus network errors and timeouts is retried with bounded backoff: 1 minute, 5 minutes, 30 minutes, then 2 hours, each with up to 20% jitter, for up to five total attempts.
- Delivery is **at-least-once** and may be out of order: deduplicate by event `id`, keep the snapshot with the greatest `data.invoice.payment_revision`, and respond `2xx` quickly (do the work after acknowledging).

## Going live

Once the loop in the [Quickstart](#quickstart) works against your test webhook (a tunnel like ngrok or cloudflared works for local dev):

1. Create an `sk_live_` key in the dashboard.
2. Set your live webhook URL in the dashboard.
3. Switch the key in your server config. Nothing else changes: same endpoints, same shapes — live invoices now carry real `payment_options`.

Test invoices and test payments never touch a chain and are never counted as real payments.

## Support

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
