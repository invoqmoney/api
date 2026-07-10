# invoq REST API

Stablecoin payments for indie developers. Non-custodial — funds settle straight to your wallet.

This is the reference for invoq's public REST API. If you use one of the official SDKs ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)), they wrap exactly these endpoints — this document is the contract they follow.

- **Base URL:** `https://api.invoq.money`
- **Hosted checkout:** `https://pay.invoq.money/<invoice id>`
- **Dashboard** (API keys, receiving wallet, webhooks): `https://app.invoq.money`

## How it works

1. **Create an invoice** from your server (`POST /v1/invoices`).
2. **Let the buyer pay it.** Easiest: send them to the hosted checkout at `https://pay.invoq.money/<invoice id>` — it shows the amount, address, and QR code, speaks ten languages, and needs zero UI work from you. Or embed the same checkout in your own site with [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). The buyer sends USDC or USDT from any wallet or exchange.
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

Take the `id` from the response and open `https://pay.invoq.money/<id>` — that's your checkout page. Since this is a test invoice, simulate the payment instead of sending real funds:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Your test webhook URL receives `invoice.paid`. That's the whole loop — the rest of this document is the detail.

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
- Extra business context is returned in `meta`, such as `retry_after`, `invoice_id`, or `reference_id`.

## Create an invoice

### `POST /v1/invoices`

Creates an invoice and returns its summary and payment instructions.

```json
{
  "amount": "12.34",
  "currency": "USD",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Field | Notes |
| --- | --- |
| `amount` | Required. Decimal string, `0.01`–`999.99`, up to 2 fractional digits for USD. Normalized in responses (`12.34` → `12.3400`). |
| `currency` | Optional. Currently only `USD` (the default). |
| `reference_id` | Optional caller-side reference, unique per project + mode, max 200 chars. Retrying with identical terms returns the existing invoice with `200 OK`; different terms return `409 reference_id_conflict`. |
| `description` | Optional payer-visible text, max 500 chars. |
| `return_url` | Optional `http(s)` URL shown as the merchant return button after payment, max 1000 chars. Omitted → the project's default return URL is snapshotted. Explicit `null` or `""` → no return URL. |

Successful response:

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
    "deposit_address": "0x...",
    "status": "unpaid",
    "amount_due": "12.340000000000000000",
    "amount_overpaid": "0.000000000000000000",
    "monitoring_ends_at": "2026-07-07T12:34:56.000Z",
    "monitoring_status": "active",
    "direct_onchain_rails": [
      {
        "chain_namespace": "eip155",
        "chain_reference": "8453",
        "token_address": "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
        "network_label": "Base",
        "display_symbol": "USDC",
        "logo_url": null,
        "chain_logo_url": null,
        "network_fee_usd": "0.0020",
        "eta_seconds": 10
      }
    ]
  }
}
```

Semantics worth knowing:

- **Funds can't be redirected through this API.** The request cannot set the recipient address, fee configuration, or settlement contract addresses — those are snapshotted from your project and account when the invoice is created. After creation an invoice is immutable except for its payment and settlement state.
- **`direct_onchain_rails` lists the ways the buyer can pay**: one entry per network + token combination (e.g. USDC on Base), each with its typical network fee and confirmation ETA. Checkout UIs render this list as the buyer's network picker.
- **Amounts are decimal strings, never floats.** Paid/due amounts use 18 fractional digits because that's the precision of on-chain token amounts. `amount_due` is derived as `max(amount − amount_paid, 0)` and `amount_overpaid` as `max(amount_paid − amount, 0)` — read those fields instead of subtracting money yourself.
- **invoq watches the chain for payments for 30 days** after creation (`monitoring_ends_at`). `monitoring_status` is `active` or `ended`; a payment that arrives after the window needs manual reconciliation from the dashboard. Test invoices have no window (`null`).
- Test invoices return `deposit_address: null` and `direct_onchain_rails: []` — they carry no real payment instructions; payments are simulated via the test-payments endpoint below.
- Rate limits per project: live 3,000/minute and 100,000/day; test 300/minute and 10,000/day.

Error codes: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (with `amount_too_small` / `amount_too_large` field codes and `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 recipient_address_not_configured`, `409 no_enabled_direct_onchain_rails`, `413 request_body_too_large`, `422 amount_not_supported_by_direct_onchain_rails`, `429 rate_limited`, `500 server_misconfigured`.

## Read an invoice

### `GET /v1/invoices/{id}`

Returns the public invoice summary, payer-visible payment state, project branding, and payment instructions. **No API key required** — invoice ids are shareable, unguessable public ids used in payment-link URLs, so this is the endpoint payment UIs poll.

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
    "deposit_address": "0x...",
    "status": "unpaid",
    "payment_status": "confirming",
    "amount_paid": "0.000000000000000000",
    "amount_due": "12.340000000000000000",
    "amount_overpaid": "0.000000000000000000",
    "monitoring_ends_at": "2026-07-07T12:34:56.000Z",
    "monitoring_status": "active",
    "direct_onchain_rails": [ { "...": "..." } ]
  }
}
```

- `status` is the canonical invoice status backed by confirmed payment and settlement events.
- `payment_status` is a payer-facing derived status, exclusive to this endpoint: `unpaid`, `confirming`, `partially_paid`, `paid`, `settling`, `settled`, or `review_required`. It matches `status` except that live invoices with a detected pending transfer show `confirming` while the canonical `status` stays unchanged until the transfer confirms.
- `review_required` means the invoice is pending manual review. It is **not** a paid state — do not fulfill on it, even if `amount_paid` looks sufficient.
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
- Partial, full, and over payments are allowed: `partially_paid` while `0 < amount_paid < amount`, `paid` once `amount_paid >= amount`. The first transition into `paid` triggers one logical `invoice.paid` webhook.

Error codes: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhooks

Configure webhook URLs in the dashboard — test and live each have their own URL and signing secret.

### Events

**`invoice.paid`** — sent after an invoice first transitions into a paid state (`paid`, `settling`, or `settled`):

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
      "fully_paid_at": "2026-06-10T10:00:00.000Z"
    }
  }
}
```

- `review_required` never triggers `invoice.paid`. Only after review clears and the invoice actually transitions to a paid state is the event created.
- `return_url` is intentionally not included; reconcile server-side by `reference_id` and invoice id.

**`webhook.ping`** — a synthetic connectivity-check event sent from the dashboard's webhook setup, signed the same way:

```json
{
  "id": "wping_...",
  "type": "webhook.ping",
  "mode": "test",
  "created_at": "2026-06-10T10:00:00.000Z",
  "data": { "project": { "id": "proj_..." } }
}
```

### Verifying signatures

Every outbound webhook includes:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` is a Unix timestamp in seconds. `v1` is the lowercase hex HMAC-SHA256 of `<t>.<raw_body>`, keyed with the mode-specific webhook secret. Verify against the **raw request body** before parsing — re-serializing the JSON can change the bytes and invalidate the signature. Reject timestamps outside your replay-tolerance window. (The official SDKs ship a `verifyWebhook` helper.)

### Delivery and retries

- Delivery POSTs time out after 10 seconds.
- Network errors, timeouts, `408`, `429`, and `5xx` are retried with bounded backoff — 1 minute, 5 minutes, 30 minutes, then 2 hours, each with up to 20% jitter — for up to five total attempts. Redirects and other `4xx` responses are non-retryable failures.
- Delivery is **at-least-once**: handle duplicate deliveries idempotently by event `id`, and respond `2xx` quickly (do the work after acknowledging).

## Going live

Once the loop in the [Quickstart](#quickstart) works against your test webhook (a tunnel like ngrok or cloudflared works for local dev):

1. Create an `sk_live_` key in the dashboard.
2. Set your live webhook URL in the dashboard.
3. Switch the key in your server config. Nothing else changes: same endpoints, same shapes — live invoices now carry a real `deposit_address` and rails.

Test invoices and test payments never touch a chain and are never counted as real payments.

## Support

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
