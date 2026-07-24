# invoq REST API

[English](../README.md) · **Bahasa Indonesia** · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Dokumen ini terjemahan dari README bahasa Inggris; kalau ada perbedaan, [versi bahasa Inggris](../README.md) yang berlaku.

Pembayaran stablecoin untuk developer indie. Non-kustodial — dana masuk langsung ke dompet Anda sendiri.

Ini referensi REST API publik invoq. Kalau Anda memakai salah satu SDK resmi ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)), semua SDK itu membungkus persis endpoint-endpoint ini — dokumen inilah kontrak yang mereka ikuti.

- **Base URL:** `https://api.invoq.money`
- **Checkout ter-hosting:** `https://pay.invoq.money/<id invoice>`
- **Dashboard** (kunci API, dompet penerima, webhook): `https://app.invoq.money`

## Cara kerjanya

1. **Buat invoice** dari server Anda (`POST /v1/invoices`).
2. **Biarkan pembeli membayarnya.** Paling mudah: arahkan mereka ke checkout ter-hosting di `https://pay.invoq.money/<id invoice>` — halaman ini menampilkan jumlah, alamat, dan kode QR, tersedia dalam sepuluh bahasa, dan tidak butuh kerja UI apa pun dari Anda. Atau tanamkan checkout yang sama di situs Anda sendiri dengan [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). Pembeli mengirim USDC atau USDT dari dompet atau bursa mana pun.
3. **Dapatkan kabar begitu dibayar.** invoq mengonfirmasi transfernya secara on-chain dan mengirim webhook `invoice.paid` ke server Anda; settlement masuk langsung ke dompet Anda sendiri.

## Mulai cepat

Ambil kunci uji coba (`sk_test_...`) dari dashboard, lalu buat invoice pertama Anda:

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

Untuk invoice produksi, `id` dari respons itu saja yang Anda butuhkan — `https://pay.invoq.money/<id>` adalah halaman checkout Anda. Invoice yang ini invoice uji coba, tanpa instruksi pembayaran on-chain, jadi simulasikan saja pembayarannya:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Kalau Anda sudah menyetel URL webhook uji coba di dashboard, URL itu menerima `invoice.paid` bertanda tangan. Itulah seluruh alurnya — sisa dokumen ini tinggal detailnya.

## Autentikasi

Endpoint server memakai kunci API rahasia yang dibuat di dashboard:

```http
Authorization: Bearer sk_test_...
```

- Kunci `sk_test_...` membuat **invoice uji coba**: pembayaran disimulasikan, tidak ada yang berpindah on-chain, dan webhook-nya sungguhan (ditandatangani dan dikirim seperti webhook produksi).
- Kunci `sk_live_...` membuat **invoice produksi** dengan instruksi pembayaran on-chain sungguhan.

Mode invoice selalu berasal dari kuncinya — isi request tidak pernah menerima field `mode`. Simpan kunci rahasia di server Anda; jangan pernah menyertakannya di kode klien.

## Struktur respons

Respons sukses membungkus resource di dalam `data`:

```json
{ "data": { "id": "inv_..." } }
```

Respons error memakai satu bentuk yang sama di semua endpoint:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` adalah kode error stabil yang bisa dibaca mesin — cabangkan logika pada kode ini, bukan pada `message`.
- `fields` hanya ada untuk error validasi tingkat field.
- Konteks bisnis tambahan dikembalikan di `meta`, misalnya `retry_after`, `reason_codes`, atau `min_amount`.

Isi request dibatasi 4KB; isi yang melebihi batas mengembalikan `413 request_body_too_large`.

## Membuat invoice

### `POST /v1/invoices`

Membuat invoice dan mengembalikan ringkasan serta instruksi pembayarannya.

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Field | Catatan |
| --- | --- |
| `amount` | Wajib. String desimal, `0.01`–`1000000.00`, maksimal 2 angka di belakang koma. Dinormalkan di respons (`12.34` → `12.3400`). Mata uang bisnisnya tetap `USD` dan dikembalikan di respons; ini bukan field request. |
| `reference_id` | Opsional, referensi sisi pemanggil, unik per proyek + mode, maksimal 200 karakter. Mengulang dengan ketentuan yang sama persis mengembalikan invoice yang sudah ada dengan `200 OK`; ketentuan yang berbeda mengembalikan `409 reference_id_conflict`. |
| `description` | Opsional, teks yang dilihat pembayar, maksimal 500 karakter. |
| `return_url` | Opsional, URL `http(s)` yang ditampilkan sebagai tombol kembali ke merchant setelah pembayaran, maksimal 1000 karakter. Dihilangkan → URL kembali bawaan proyek di-snapshot. `null` atau `""` eksplisit → tanpa URL kembali. Pada pengulangan `reference_id`, `return_url` yang dihilangkan tidak divalidasi terhadap invoice yang sudah ada — kirimkan secara eksplisit kalau pengulangan itu harus menegaskan nilai tertentu. |

Respons sukses (`201 Created`; pemakaian ulang idempoten mengembalikan `200 OK` dengan `meta.result: "reused"`):

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

Semantik yang perlu diketahui:

- **Dana tidak bisa dialihkan lewat API ini.** Request tidak bisa menyetel alamat penerima atau konfigurasi kontrak — semuanya berasal dari pengaturan proyek terverifikasi Anda saat invoice dibuat. Setelah dibuat, invoice tidak bisa diubah lagi kecuali status pembayaran dan settlement-nya.
- **`payment_options` mendaftar cara-cara pembeli bisa membayar**, satu entri per jaringan + token, dalam urutan yang ditampilkan ke pembayar (USDT sebelum USDC, lalu urutan jaringan hasil peninjauan untuk tiap token). `collection_method` adalah pembedanya:
  - `evm_deposit` — alamat deposit EVM per invoice. Setiap transfer positif yang tiba tepat waktu dikreditkan sebesar jumlahnya; `suggested_amount` (`max(0, amount_due − pending)`) hanyalah panduan, bukan syarat pencocokan.
  - `direct_exact` — alamat merchant Solana/TRON dengan jumlah eksak. Pembeli harus mengirim tepat `exact_amount` (`invoice_amount + matching_increment`) dalam satu transfer; increment itulah cara pembayaran diatribusikan dan tidak pernah menjadi kredit invoice.
  - Hanya opsi ber-`status: "ready"` yang membawa field pembayaran di atas; opsi `unavailable` hanya membawa field umumnya. Identifikasi sebuah opsi lewat (`chain_namespace`, `chain_reference`, `token_address`), jangan pernah lewat posisi array atau metadata tampilan.
- **`checkout_status` adalah status yang dilihat pembayar**, diturunkan pada setiap respons: `paid` (status kanonisnya paid/settling/settled), `confirming` (menunggu bukti on-chain), `expired` (melewati `monitoring_ends_at`), `open` (minimal satu opsi siap), atau `unavailable`. Status ini tidak pernah menjadi izin memproses pesanan — gunakan webhook `invoice.paid`.
- **`payment_revision`** mulai dari `0` dan bertambah tepat satu kali setiap kali himpunan pembayaran terkonfirmasi yang dikreditkan berubah (termasuk tiap pembayaran uji coba baru). Gunakan untuk membuang snapshot invoice atau webhook lama yang tiba setelah yang lebih baru.
- **Jumlah adalah string desimal, tidak pernah float.** Jumlah terbayar/terutang memakai 18 angka di belakang koma. `amount_due` adalah `max(amount − amount_paid, 0)` dan `amount_overpaid` adalah `max(amount_paid − amount, 0)` — baca kedua field itu, jangan mengurangkan uang sendiri.
- **invoq memantau chain selama 7 hari** setelah pembuatan (`monitoring_ends_at`). Transfer yang tiba tepat pada atau setelah saat itu tetap dicatat tetapi tidak mengkredit apa pun; rekonsiliasi manual di dashboard adalah jaring pengaman operator untuk edge case di dalam jendela pemantauan.
- Invoice uji coba mengembalikan `monitoring_ends_at: null`, `payment_options: []`, dan `checkout_status: "unavailable"` — pembayaran disimulasikan lewat endpoint test-payments di bawah.
- Rate limit per proyek: produksi 3.000/menit dan 100.000/hari; uji coba 300/menit dan 10.000/hari.

Kode error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (dengan kode field `amount_too_small` / `amount_too_large` serta `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (dengan `meta.reason_codes` terurut: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — `merchant_address_provisioning` bersifat sementara dan biasanya hilang dalam satu-dua menit), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Membaca invoice

### `GET /v1/invoices/{id}`

Mengembalikan ringkasan invoice publik, status pembayaran yang dilihat pembayar, branding proyek, dan instruksi pembayaran. **Tidak butuh kunci API** — id invoice adalah id publik yang bisa dibagikan dan tidak bisa ditebak, dipakai di URL tautan pembayaran, jadi endpoint inilah yang di-poll UI pembayaran (CORS mengizinkan origin mana pun untuk GET).

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

- `status` adalah status invoice kanonis yang didukung event pembayaran dan settlement terkonfirmasi: `unpaid`, `partially_paid`, `paid`, `settling`, `settled`, atau `review_required`.
- `review_required` berarti invoice sedang menunggu peninjauan manual. Ini **bukan** status terbayar — jangan proses pesanan karenanya, meskipun `amount_paid` terlihat cukup.
- `checkout_status` adalah status turunan yang dilihat pembayar seperti dijelaskan di atas; invoice produksi dengan bukti on-chain yang masih menunggu menampilkan `confirming`, sementara `status` kanonis tidak berubah sampai transfernya terkonfirmasi.
- `transfers` adalah jejak penerimaan yang dilihat pembayar: transfer masuk terkonfirmasi yang mengkredit invoice ini, sehingga checkout bisa menampilkan tiap transaksi on-chain dan menautkan block explorer. Tiap entri membawa `chain_namespace`, `chain_reference`, `transaction_id` kanonis, `event_index`, `amount` (dalam satuan mata uang invoice pada skala 18 desimal yang sama dengan `amount_paid`; untuk `direct_exact` tidak termasuk matching increment), dan `explorer_transaction_url` (atau `null`). Hanya transfer terkonfirmasi yang muncul — transfer yang masih pending bisa saja hilang karena reorg — dibatasi 20 terbesar berdasarkan jumlah supaya dust yang dikirim ke alamat deposit publik tidak menenggelamkan pembayaran sungguhan. Selalu ada: `[]` sampai ada transfer yang terkonfirmasi, dan selalu `[]` untuk invoice uji coba.
- `reference_id` yang hanya untuk pemanggil tidak disertakan di sini; hanya field branding `project` untuk pembayar yang dikembalikan.
- Id dengan bentuk tidak valid maupun id yang tidak dikenal sama-sama mengembalikan `404 invoice_not_found`.

## Menyimulasikan pembayaran uji coba

### `POST /v1/invoices/{id}/test-payments`

Menambahkan pembayaran simulasi ke invoice **uji coba** dan mengembalikan status pembayaran terbaru. Hanya tersedia dengan kunci `sk_test_...` — inilah cara Anda menguji alur penuh unpaid → paid → webhook tanpa menyentuh chain sama sekali.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` wajib, harus lebih besar dari nol, maksimal 15 digit di depan dan 4 digit di belakang koma (`5`, `5.0`, `5.0000` dinormalkan menjadi `5.0000`).
- `reference_id` opsional, maksimal 200 karakter, dan idempoten per invoice: memakainya lagi dengan jumlah ternormalkan yang sama mengembalikan `200 OK` dengan `meta.result: "reused"`; jumlah yang berbeda mengembalikan `409 test_payment_reference_conflict`.
- Pembayaran parsial, penuh, dan berlebih semuanya diperbolehkan: `partially_paid` selama `0 < amount_paid < amount`, `paid` begitu `amount_paid >= amount`. Transisi pertama ke `paid` memicu satu webhook `invoice.paid` logis, dan tiap pembayaran yang dibuat menambah `payment_revision`.
- Pembuatan dibatasi 300 per menit dan 10.000 per hari per proyek.

Kode error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhook

Atur URL webhook di dashboard — mode uji coba dan produksi masing-masing punya URL dan kunci rahasia penandatanganan sendiri.

### Event

**`invoice.paid`** — dikirim saat invoice bertransisi ke status terbayar (`paid`, `settling`, atau `settled`):

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

**`invoice.payment_reversed`** — dikirim saat invoice yang sebelumnya terbayar turun lagi ke bawah jumlahnya (misalnya reorg chain menghapus transfer yang sudah dikreditkan). Bentuk payload-nya sama, dengan `status` invoice saat ini, `amount_paid`, `payment_revision` yang lebih tinggi, dan `fully_paid_at: null`. Perlakukan event ini sebagai pembatalan sinyal pemrosesan pesanan yang lebih awal, sesuai kebijakan bisnis Anda sendiri.

- `review_required` tidak pernah memicu `invoice.paid`. Event-nya baru dibuat setelah peninjauan selesai dan invoice masuk ke status terbayar.
- Urutan paid → reversed → paid yang sungguhan mengirimkan `invoice.paid`, `invoice.payment_reversed`, lalu `invoice.paid` baru, masing-masing dengan `payment_revision` hasilnya.
- `reference_id` dan `fully_paid_at` bisa bernilai null tetapi selalu ada; `return_url` dan instruksi pembayaran sengaja tidak disertakan. Rekonsiliasikan di sisi server lewat id invoice plus `reference_id`.

### Memverifikasi tanda tangan

Setiap webhook keluar menyertakan:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` adalah timestamp Unix dalam detik. `v1` adalah HMAC-SHA256 heks huruf kecil dari `<t>.<raw_body>`, memakai kunci rahasia webhook untuk mode itu sebagai kuncinya. Verifikasi terhadap **isi request mentah** sebelum parsing — menyerialisasi ulang JSON bisa mengubah byte-nya dan membuat tanda tangan tidak valid. Tolak timestamp di luar jendela toleransi replay Anda. (SDK resmi menyertakan helper `verifyWebhook`.)

### Pengiriman dan percobaan ulang

- POST pengiriman timeout setelah 10 detik.
- Setiap respons non-2xx — **termasuk redirect dan 4xx** — plus error jaringan dan timeout akan diulang dengan backoff terbatas: 1 menit, 5 menit, 30 menit, lalu 2 jam, masing-masing dengan jitter hingga 20%, sampai total lima percobaan.
- Pengiriman bersifat **at-least-once** dan bisa tidak berurutan: deduplikasi berdasarkan `id` event, simpan snapshot dengan `data.invoice.payment_revision` terbesar, dan balas `2xx` secepatnya (kerjakan prosesnya setelah membalas).

## Masuk produksi

Begitu alur di [Mulai cepat](#mulai-cepat) berjalan terhadap webhook uji coba Anda (tunnel seperti ngrok atau cloudflared cocok untuk pengembangan lokal):

1. Buat kunci `sk_live_` di dashboard.
2. Setel URL webhook produksi Anda di dashboard.
3. Ganti kuncinya di konfigurasi server Anda. Tidak ada lagi yang berubah: endpoint sama, bentuk sama — invoice produksi kini membawa `payment_options` sungguhan.

Invoice uji coba dan pembayaran uji coba tidak pernah menyentuh chain dan tidak pernah dihitung sebagai pembayaran sungguhan.

## Dukungan

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
