# invoq REST API

[English](../README.md) · **Bahasa Indonesia** · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Dokumen ini terjemahan dari README bahasa Inggris; kalau ada perbedaan, [versi bahasa Inggris](../README.md) yang berlaku.

Pembayaran stablecoin, terintegrasi ke produk Anda. Non-kustodial — dana langsung masuk ke dompet Anda sendiri.

Ini referensi REST API publik invoq. SDK resmi — [Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby) — membungkus persis endpoint-endpoint ini.

- **Base URL:** `https://api.invoq.money`
- **Checkout ter-hosting:** `https://pay.invoq.money/<id invoice>`
- **Dashboard** (kunci API, dompet penerima, webhook): `https://app.invoq.money`
- **OpenAPI 3.1:** `https://api.invoq.money/openapi.json` — kontrak ini, dalam bentuk yang bisa dibaca mesin

**Coding pakai AI? Tempelkan ini.**

```
Tambahkan pembayaran stablecoin ke proyek saya dengan invoq. Mulai dari mode tes. Baca dokumentasinya sebelum menulis kode: https://invoq.money/llms.txt
```

## Cara kerjanya

1. **Buat invoice** dari server Anda (`POST /v1/invoices`).
2. **Biarkan pembeli membayarnya.** Cara termudah: arahkan mereka ke `https://pay.invoq.money/<id invoice>` — halamannya menampilkan jumlah, alamat, dan QR code, berbicara sepuluh bahasa, dan tidak menuntut UI apa pun dari Anda. Atau sematkan checkout yang sama dengan [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). Pembeli mengirim USDT atau USDC dari dompet atau exchange mana pun.
3. **Dapat kabar begitu terbayar.** invoq mengonfirmasi transfer on-chain lalu mengirim webhook `invoice.paid` ke server Anda. Dananya langsung mengalir ke dompet Anda sendiri.

## Mulai cepat

Ambil kunci uji coba (`sk_test_...`) di dashboard, lalu buat invoice pertama Anda:

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

Untuk invoice produksi, `id` di responsnya sudah cukup — `https://pay.invoq.money/<id>` adalah halaman pembayaran Anda. Yang di atas ini invoice uji coba, yang tidak membawa instruksi pembayaran on-chain, jadi simulasikan pembayarannya:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Kalau Anda sudah menyetel URL webhook uji coba di dashboard, URL itu akan menerima `invoice.paid` bertanda tangan. Itu seluruh alurnya — sisa dokumen ini adalah detailnya.

## Autentikasi

Endpoint server memakai kunci API rahasia yang dibuat di dashboard:

```http
Authorization: Bearer sk_test_...
```

- Kunci `sk_test_...` membuat **invoice uji coba**: pembayarannya disimulasikan, tidak ada yang bergerak on-chain, dan webhook-nya sungguhan — ditandatangani dan dikirim persis seperti yang produksi.
- Kunci `sk_live_...` membuat **invoice produksi** dengan instruksi pembayaran on-chain sungguhan.

Mode selalu berasal dari kunci; body request tidak pernah menerima field `mode`. Simpan kunci rahasia di server Anda, jangan pernah di kode klien.

`GET /v1/invoices/{id}` tidak butuh kunci sama sekali. Id invoice adalah id publik yang tidak bisa ditebak dan dipakai di tautan pembayaran, jadi UI checkout bisa mem-polling-nya langsung dari browser — CORS mengizinkan origin mana pun untuk `GET` dan `HEAD`.

## Request dan respons

Respons yang berhasil membungkus resource di dalam `data`:

```json
{ "data": { "id": "inv_..." } }
```

Semua endpoint memakai satu bentuk error yang sama:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- Bercabanglah berdasarkan `code`. `message` untuk dibaca manusia dan bisa berubah kapan saja.
- `fields` hanya muncul pada error validasi per field; `location` bernilai `body`, `query`, atau `path`.
- Konteks bisnis tambahan masuk ke `meta` — `retry_after`, `reason_codes`, `min_amount`, dan sejenisnya.

Beberapa konvensi yang sebaiknya diketahui:

- **Validasinya ketat.** Kunci body atau parameter query yang tidak dikenal menghasilkan `400 invalid_request` dengan `fields[].code: "unknown_field"` — menempelkan parameter anti-cache membuat seluruh request gagal.
- **Nominal selalu string desimal, tidak pernah float.** Nominal invoice memakai 4 angka desimal, nominal terbayar dan sisa 18, dan nominal token di dalam `payment_options` persis sebanyak `token_decimals`.
- **`429 rate_limited` menaruh petunjuknya di body**, sebagai `meta.retry_after` dalam detik bulat. Tidak ada header `Retry-After` yang dikirim — bacalah `meta`.
- Body request dibatasi 4KB; lebih dari itu menjadi `413 request_body_too_large`.
- Setiap respons JSON membawa `Cache-Control: no-store` — status pembayaran di-polling, jadi tidak boleh ada yang menyodorkan invoice basi.
- `GET /` adalah liveness probe tanpa autentikasi. Ia mengembalikan `204 No Content`.
- `GET /openapi.json` menyajikan kontrak ini — ketiga endpoint dan kedua webhook — sebagai OpenAPI 3.1. Dokumen itu dibangun dari skema yang dipakai API untuk memvalidasi, jadi tidak mungkin menggambarkan server yang berbeda. Pakai untuk membuat klien di bahasa yang belum punya SDK.

Pembuatan invoice dibatasi per proyek: produksi 3.000/menit dan 100.000/hari, uji coba 300/menit dan 10.000/hari.

## Membuat invoice

### `POST /v1/invoices`

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
| `amount` | Wajib. String desimal, `0.01`–`1000000.00`, maksimal 2 angka desimal. Dinormalisasi di respons (`12.34` → `12.3400`). Mata uangnya tetap `USD` dan dikembalikan di respons; ini bukan field request. |
| `reference_id` | Referensi Anda sendiri, opsional, unik per proyek dan mode, maksimal 200 karakter. Mengulang dengan syarat yang sama mengembalikan invoice yang sudah ada dengan `200 OK` dan `meta.result: "reused"`; syarat berbeda mengembalikan `409 reference_id_conflict`. |
| `description` | Teks opsional yang terlihat oleh pembayar, maksimal 500 karakter. |
| `return_url` | URL `http(s)` opsional, maksimal 1000 karakter — tombol kembali ke merchant yang ditampilkan setelah pembayaran. Jika dihilangkan, nilai default proyek di-snapshot ke invoice; kirim `null` atau `""` kalau tidak mau ada URL kembali. Pada pengulangan dengan `reference_id`, `return_url` yang dihilangkan tidak dicek terhadap invoice yang ada, jadi kirimkan secara eksplisit bila pengulangan itu harus menegaskan suatu nilai. |

`201 Created`, atau `200 OK` pada pemakaian ulang idempoten:

> SDK resmi hanya mengembalikan resource-nya dan membuang `meta.result`, jadi lewat SDK sebuah create dan pemakaian ulang idempoten tidak bisa dibedakan — dan itulah gunanya `reference_id`. Sandarkan pembukuanmu pada `reference_id` yang kamu kirim; panggil endpoint-nya langsung kalau kamu butuh pembedaan itu.

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

- **Dana tidak bisa dialihkan lewat API ini.** Request tidak bisa menyetel alamat penerima atau konfigurasi kontrak — semuanya berasal dari pengaturan proyek terverifikasi Anda. Setelah dibuat, invoice tidak bisa diubah lagi kecuali status pembayaran dan settlement-nya.
- Invoice uji coba mengembalikan `monitoring_ends_at: null`, `payment_options: []`, dan `checkout_status: "unavailable"`. Bayar lewat [endpoint test-payments](#post-v1invoicesidtest-payments).

Kode error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (kode field `invalid_format`, atau `amount_too_small` / `amount_too_large` dengan `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

`409 no_payment_options_available` berarti tidak ada opsi pembayaran yang bisa diterbitkan, dan membawa `meta.reason_codes` yang terurut: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted`. `merchant_address_provisioning` sifatnya sementara — alamat Solana atau TRON yang baru masih diaktifkan, dan request yang sama biasanya berhasil beberapa detik kemudian.

## Membaca invoice

### `GET /v1/invoices/{id}`

Tampilan invoice dari sisi pembayar: ringkasan, status pembayaran, identitas proyek, instruksi pembayaran, dan riwayat transfer yang masuk. Tidak butuh kunci API — inilah endpoint yang di-polling UI checkout.

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

Dibanding respons pembuatan, di sini ada tambahan `amount_paid`, `project`, dan `transfers`, sementara `reference_id` milik Anda tidak dikembalikan. `description`, `return_url`, `project.name`, dan `project.logo_url` semuanya bisa `null` — checkout harus tetap bisa tampil tanpa satu pun dari semuanya.

`transfers` adalah riwayat penerimaan, supaya checkout bisa menampilkan tiap transaksi on-chain dan menautkannya ke block explorer. Tiap entri membawa `chain_namespace`, `chain_reference`, `transaction_id`, `event_index`, `amount` (dalam mata uang invoice, pada skala 18 desimal yang sama dengan `amount_paid`; untuk `direct_exact` tidak termasuk increment pencocokan), dan `explorer_transaction_url` (`null` bila tidak ada yang dikonfigurasi).

Hanya transfer terkonfirmasi yang muncul — yang masih pending bisa saja hilang karena reorg — dibatasi 20 transfer terbesar berdasarkan nominal, dari yang terbesar, supaya debu tidak menggeser pembayaran yang sebenarnya. Selalu ada: `[]` sampai ada transfer yang terkonfirmasi, dan selalu `[]` untuk invoice uji coba.

Id yang bentuknya salah maupun yang tidak dikenal sama-sama mengembalikan `404 invoice_not_found`.

## Bagaimana pembeli membayar

`payment_options` mendaftar cara-cara membayar invoice ini, satu entri per jaringan dan token, dalam urutan yang ditampilkan ke pembayar: USDT sebelum USDC, lalu urutan jaringan tiap token. Opsi-opsinya, beserta alamat dan nominalnya, dikunci saat invoice dibuat — alamat penerima atau jalur yang Anda konfigurasi belakangan tidak menulis ulang invoice yang sudah ada. Hanya `status` tiap opsi yang dihitung ulang pada setiap respons.

Identifikasi sebuah opsi lewat (`chain_namespace`, `chain_reference`, `token_address`). Jangan pernah lewat posisi array, dan jangan pernah lewat `network_label`, `display_symbol`, `logo_url`, atau `chain_logo_url`, yang hanya metadata tampilan.

`collection_method` adalah pembedanya.

**`evm_deposit`** — alamat deposit yang hanya milik invoice ini:

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

Setiap transfer positif yang tiba tepat waktu dikreditkan sebesar nominalnya. `suggested_amount` hanyalah panduan, bukan syarat pencocokan: nilainya `max(0, amount_due − pending)` yang dibulatkan **ke atas** ke `token_decimals`, atau ke 6 digit bila jalur itu membawa lebih — angka ini diketik ulang oleh manusia — lalu ditambahi nol sampai `token_decimals`. Jadi bisa melebihi `amount_due` sampai `0.000001`. Jangan berasumsi keduanya sama.

**`direct_exact`** — alamat Solana atau TRON Anda ditambah satu nominal eksak:

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

Pembeli harus mengirim tepat `exact_amount` (`invoice_amount + matching_increment`) dalam satu transfer. Increment itulah cara pembayaran diatribusikan ke invoice ini; uangnya sampai ke Anda, tetapi tidak pernah menjadi kredit invoice.

Karena nominal eksak itu selalu menutup seluruh invoice, opsi `direct_exact` berubah menjadi `unavailable` begitu invoice punya pembayaran terkonfirmasi atau pending mana pun, termasuk yang datang lewat jalur lain. Karena alasan yang sama, invoice direct yang baru terbayar sebagian tidak bisa ditambal: terbitkan invoice baru untuk sisanya.

Hanya opsi ber-`status: "ready"` yang membawa field pembayaran di atas. Opsi `unavailable` hanya membawa field umum: ia sudah tidak beroperasi (peninjauan manual, alamat atau jalur diblokir, chain sedang dijeda, jendela pembayaran sudah lewat) dan sebaiknya tidak lagi ditawarkan ke pembeli.

## Status pembayaran

Ada dua field status, dan keduanya menjawab pertanyaan berbeda.

**`status`** adalah status pembukuan kanonis, disokong pembayaran terkonfirmasi dan settlement: `unpaid`, `partially_paid`, `paid`, `settling`, `settled`, atau `review_required`. `paid`, `settling`, dan `settled` sama-sama berarti pembeli sudah membayar — bedanya hanya sejauh mana dananya bergerak ke dompet Anda. `review_required` berarti invoice sedang menunggu peninjauan manual — itu **bukan** status terbayar, jadi jangan memproses pesanan sekalipun `amount_paid` terlihat cukup.

**`checkout_status`** adalah status yang dilihat pembayar, diturunkan pada setiap respons dan dievaluasi dengan urutan berikut:

| Nilai | Artinya |
| --- | --- |
| `paid` | `status` bernilai `paid`, `settling`, atau `settled` |
| `confirming` | Bukti on-chain sudah datang, belum terkonfirmasi |
| `expired` | Sudah melewati `monitoring_ends_at` |
| `open` | Ada minimal satu opsi pembayaran ber-`ready` |
| `unavailable` | Selain itu: peninjauan, jalur diblokir, invoice uji coba yang belum dibayar |

**`checkout_status` tidak pernah menjadi izin memproses pesanan.** Gunakan webhook `invoice.paid`.

**`payment_revision`** mulai dari `0` dan bertambah satu setiap kali himpunan pembayaran terkonfirmasi yang dikreditkan berubah: transfer baru, pembalikan, pembayaran uji coba baru, atau koreksi waktu on-chain sebuah transfer yang sudah dikreditkan. Settlement saja tidak menggerakkannya, dan nilainya bisa berubah walau `status` tidak. Gunakan untuk membuang snapshot invoice atau webhook yang tiba setelah yang lebih baru.

`amount_due` adalah `max(amount − amount_paid, 0)` dan `amount_overpaid` adalah `max(amount_paid − amount, 0)`. Bacalah field itu alih-alih mengurangkan nominal sendiri.

## Jendela pembayaran

`monitoring_ends_at` jatuh satu hari setelah invoice dibuat, dan itulah satu-satunya batas. Sebuah transfer hanya dikreditkan otomatis bila waktu on-chain transfer itu jatuh di dalam jendela — tidak ada yang dari sebelum invoice ada, tidak ada yang tepat di `monitoring_ends_at` atau sesudahnya. Jam Anda, waktu kami mengamati, dan waktu tibanya webhook tidak ikut menentukan.

Pembayaran yang telat tidak hilang. Ia tetap dicatat pada invoice dan terlihat di dashboard, tempat Anda bisa mengkreditkannya dengan menyebut transaksinya — sebuah penegasan yang tidak bisa dibuat oleh proses otomatis mana pun. Berapa lama waktunya bergantung pada jalurnya:

- **EVM** — tanpa tenggat. Alamat deposit hanya milik invoice ini dan tidak pernah diterbitkan ulang, jadi transfer yang sampai ke sana tidak mungkin membayar hal lain.
- **Solana dan TRON** — 21 hari sejak dibuat. Nominal eksaknya tetap dicadangkan 20 hari lagi setelah `monitoring_ends_at`; lewat dari itu nominal tersebut bisa jadi milik invoice yang lebih baru, dan tak seorang pun bisa memastikan mana yang dibayar sebuah transfer terlambat. Yang dihitung adalah kapan transfernya tiba, bukan kapan Anda sempat mengurusnya.

Satu konsekuensinya untuk integrasi Anda: **`invoice.paid` bisa tiba jauh setelah checkout menampilkan `expired`**, dan tidak ada field yang membedakannya. Kalau Anda membatalkan atau menjual ulang pesanan begitu checkout-nya kedaluwarsa, cocokkan `invoice.paid` dengan status pesanan Anda sendiri alih-alih mengira invoice-nya masih terbuka — dan tetap proses secara idempoten dalam kondisi apa pun.

## Webhook

Atur URL webhook di dashboard. Uji coba dan produksi masing-masing punya URL dan secret penanda tangan sendiri.

### Event

**`invoice.paid`** — invoice mencapai status terbayar (`paid`, `settling`, atau `settled`):

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

**`invoice.payment_reversed`** — invoice yang tadinya terbayar turun lagi di bawah nominalnya, misalnya karena reorg menghapus transfer yang sudah dikreditkan. Bentuk payload-nya sama, dengan `status` invoice saat ini, `payment_revision` yang lebih tinggi, dan `fully_paid_at: null`. Perlakukan sebagai pembatalan sinyal pemenuhan sebelumnya, sesuai kebijakan bisnis Anda sendiri.

- `review_required` tidak pernah memicu `invoice.paid`. Kalau ambangnya terlewati saat peninjauan, event-nya dikirim satu kali saja, setelah peninjauan selesai.
- Urutan nyata terbayar → dibalik → terbayar mengirim `invoice.paid`, lalu `invoice.payment_reversed`, lalu `invoice.paid` yang baru, masing-masing dengan `payment_revision` hasilnya.
- `reference_id` dan `fully_paid_at` bisa `null` tetapi selalu ada. `return_url` dan instruksi pembayaran sengaja tidak disertakan — rekonsiliasikan di sisi server dengan id invoice plus `reference_id`.

### Memverifikasi tanda tangan

Tiap pengiriman membawa:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` adalah timestamp Unix dalam detik. `v1` adalah HMAC-SHA256 heksadesimal huruf kecil dari `<t>.<raw_body>`, memakai secret webhook mode terkait. Verifikasi terhadap **body mentah** sebelum mem-parsing — menserialisasi ulang JSON bisa mengubah byte-nya dan membuat tanda tangan tidak valid. Tolak timestamp di luar jendela toleransi replay Anda. SDK resmi menyertakan helper `verifyWebhook`.

### Pengiriman dan percobaan ulang

- Pengiriman timeout setelah 10 detik.
- Semua respons non-2xx — **termasuk redirect dan 4xx** — plus error jaringan dan timeout akan dicoba ulang dengan backoff terbatas: 1 menit, 5 menit, 30 menit, lalu 2 jam, masing-masing dengan jitter sampai 20%, sebanyak lima percobaan total.
- Pengiriman bersifat **at-least-once dan bisa tidak berurutan.** Deduplikasi berdasarkan `id` event, simpan snapshot dengan `payment_revision` tertinggi, dan balas `2xx` dengan cepat — kerjakan prosesnya setelah memberi tanda terima.

## Mode uji coba dan masuk produksi

### `POST /v1/invoices/{id}/test-payments`

Menambahkan pembayaran simulasi ke invoice **uji coba** dan mengembalikan status pembayarannya yang terbaru. Hanya tersedia dengan kunci `sk_test_...` — beginilah cara Anda menjalankan seluruh alur belum dibayar → terbayar → webhook tanpa menyentuh chain.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` wajib dan lebih besar dari nol, sampai 15 digit bulat dan 4 angka desimal (`5`, `5.0`, dan `5.0000` sama-sama dinormalkan ke `5.0000`).
- `reference_id` opsional, maksimal 200 karakter, dan idempoten per invoice: referensi yang sama dengan nominal ternormalisasi yang sama mengembalikan `200 OK` dan `meta.result: "reused"`; nominal berbeda mengembalikan `409 test_payment_reference_conflict`.
- Pembayaran sebagian, penuh, dan lebih semuanya diperbolehkan: `partially_paid` selama `0 < amount_paid < amount`, `paid` begitu `amount_paid >= amount`. Perlintasan pertama ke `paid` mengirim satu `invoice.paid`, dan tiap pembayaran baru menaikkan `payment_revision`.
- Dibatasi 300 per menit dan 10.000 per hari per proyek.

Responsnya adalah invoice dalam bentuk seperti saat dibuat, plus `amount_paid` dan `fully_paid_at`, disertai `meta.result`.

Kode error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

### Masuk produksi

Begitu alurnya berjalan terhadap webhook uji coba Anda — tunnel seperti ngrok atau cloudflared cocok untuk pengembangan lokal:

1. Buat kunci `sk_live_` di dashboard.
2. Setel URL webhook produksi Anda.
3. Ganti kuncinya di konfigurasi server Anda.

Tidak ada lagi yang berubah: endpoint sama, bentuk sama, dan invoice produksi kini membawa `payment_options` sungguhan. Invoice uji coba dan pembayaran uji coba tidak pernah menyentuh chain dan tidak pernah dihitung sebagai pembayaran sungguhan.

## Dukungan

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
