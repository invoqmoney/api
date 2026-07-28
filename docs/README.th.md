# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · **ไทย** · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> เอกสารนี้แปลจาก README ภาษาอังกฤษ หากมีข้อความไม่ตรงกัน ให้ยึด[ฉบับภาษาอังกฤษ](../README.md)เป็นหลัก

รับชำระเงินด้วย stablecoin เชื่อมต่อเข้ากับระบบของคุณ ไม่ถือเงินแทนคุณ — เงินเข้ากระเป๋าของคุณโดยตรง

นี่คือเอกสารอ้างอิงของ REST API สาธารณะของ invoq โดย SDK ทางการ — [Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby) — ห่อหุ้มปลายทางเหล่านี้ตรง ๆ

- **Base URL:** `https://api.invoq.money`
- **หน้าชำระเงินที่โฮสต์ให้:** `https://pay.invoq.money/<invoice id>`
- **แดชบอร์ด** (คีย์ API, กระเป๋าเงินสำหรับรับเงิน, webhook): `https://app.invoq.money`
- **OpenAPI 3.1:** `https://api.invoq.money/openapi.json` — สัญญาฉบับนี้ในรูปแบบที่เครื่องอ่านได้

**ใช้ AI เขียนโค้ดอยู่ไหม วางข้อความนี้**

```
เพิ่มการรับชำระด้วย stablecoin เข้าโปรเจกต์ของฉันด้วย invoq เริ่มที่โหมดทดสอบ อ่านเอกสารก่อนเขียนโค้ด: https://invoq.money/llms.txt
```

## หลักการทำงาน

1. **สร้างใบแจ้งหนี้** จากเซิร์ฟเวอร์ของคุณ (`POST /v1/invoices`)
2. **ให้ผู้ซื้อจ่ายเงิน** วิธีที่ง่ายที่สุดคือส่งเขาไปที่ `https://pay.invoq.money/<invoice id>` — หน้านั้นแสดงยอดเงิน ที่อยู่ และ QR code รองรับสิบภาษา และคุณไม่ต้องเขียน UI เลยสักบรรทัด หรือจะฝังหน้าเดียวกันนี้ลงเว็บของคุณด้วย [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) ก็ได้ ผู้ซื้อโอน USDT หรือ USDC มาจากกระเป๋าหรือ exchange ไหนก็ได้
3. **รู้ทันทีเมื่อจ่ายแล้ว** invoq ยืนยันการโอนบนเชนแล้วส่ง webhook `invoice.paid` ไปยังเซิร์ฟเวอร์ของคุณ เงินเข้ากระเป๋าของคุณเองโดยตรง

## เริ่มต้นอย่างรวดเร็ว

ไปเอาคีย์ทดสอบ (`sk_test_...`) จากแดชบอร์ด แล้วสร้างใบแจ้งหนี้ใบแรก:

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

สำหรับใบแจ้งหนี้จริง แค่ `id` ในผลลัพธ์ก็พอแล้ว — `https://pay.invoq.money/<id>` คือหน้าชำระเงินของคุณ ใบข้างบนเป็นใบทดสอบซึ่งไม่มีข้อมูลการชำระเงินบนเชน จึงต้องจำลองการจ่ายแทน:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

ถ้าคุณตั้ง URL webhook ทดสอบไว้ในแดชบอร์ด มันจะได้รับ `invoice.paid` ที่มีลายเซ็น เท่านี้ก็ครบวงจรแล้ว — ส่วนที่เหลือของเอกสารนี้คือรายละเอียด

## การยืนยันตัวตน

ปลายทางฝั่งเซิร์ฟเวอร์ใช้คีย์ API ลับที่สร้างจากแดชบอร์ด:

```http
Authorization: Bearer sk_test_...
```

- คีย์ `sk_test_...` สร้าง**ใบแจ้งหนี้ทดสอบ** การจ่ายเป็นการจำลอง ไม่มีอะไรเคลื่อนไหวบนเชน แต่ webhook เป็นของจริง — เซ็นและส่งเหมือนของจริงทุกอย่าง
- คีย์ `sk_live_...` สร้าง**ใบแจ้งหนี้จริง** พร้อมข้อมูลการชำระเงินบนเชนของจริง

โหมดมาจากคีย์เสมอ body ของ request ไม่เคยรับฟิลด์ `mode` เก็บคีย์ลับไว้บนเซิร์ฟเวอร์ อย่าเอาไปไว้ในโค้ดฝั่งไคลเอ็นต์

`GET /v1/invoices/{id}` ไม่ต้องใช้คีย์เลย เพราะ id ของใบแจ้งหนี้เป็น id สาธารณะที่เดาไม่ได้และใช้ในลิงก์ชำระเงินอยู่แล้ว หน้าชำระเงินจึงเรียกซ้ำ ๆ จากเบราว์เซอร์ได้โดยตรง — CORS อนุญาตทุก origin สำหรับ `GET` และ `HEAD`

## Request และการตอบกลับ

การตอบกลับที่สำเร็จจะห่อทรัพยากรไว้ใน `data`:

```json
{ "data": { "id": "inv_..." } }
```

ทุกปลายทางใช้รูปแบบข้อผิดพลาดเดียวกัน:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- ให้แยกเงื่อนไขจาก `code` ส่วน `message` มีไว้ให้คนอ่านและเปลี่ยนได้ตลอด
- `fields` จะโผล่เฉพาะตอนมีข้อผิดพลาดระดับฟิลด์ ส่วน `location` เป็น `body`, `query` หรือ `path`
- ข้อมูลเชิงธุรกิจเพิ่มเติมอยู่ใน `meta` เช่น `retry_after`, `reason_codes`, `min_amount`

ข้อตกลงบางอย่างที่ควรรู้:

- **การตรวจสอบเข้มงวด** คีย์แปลกปลอมใน body หรือ query parameter ที่ไม่รู้จักจะได้ `400 invalid_request` พร้อม `fields[].code: "unknown_field"` — แค่ต่อท้าย parameter กัน cache ก็ทำให้ทั้ง request ล้มเหลว
- **ยอดเงินเป็นสตริงทศนิยมเสมอ ไม่เคยเป็น float** ยอดของใบแจ้งหนี้มีทศนิยม 4 ตำแหน่ง ยอดที่จ่ายแล้วและยอดคงเหลือมี 18 ตำแหน่ง ส่วนยอดแบบโทเคนใน `payment_options` มีเท่ากับ `token_decimals` พอดี
- **`429 rate_limited` ใส่คำใบ้ไว้ใน body** คือ `meta.retry_after` เป็นวินาทีเต็ม ไม่มีการส่ง header `Retry-After` — ให้อ่านจาก `meta`
- body ของ request จำกัดที่ 4KB เกินกว่านั้นจะเป็น `413 request_body_too_large`
- ทุกการตอบกลับแบบ JSON มี `Cache-Control: no-store` — สถานะการชำระเงินต้องถูก poll จึงต้องไม่มีใครส่งใบแจ้งหนี้เก่าค้างมาให้คุณ
- `GET /` คือปลายทางตรวจสถานะที่ไม่ต้องยืนยันตัวตน ตอบกลับ `204 No Content`
- `GET /openapi.json` ให้สัญญาฉบับนี้ — ทั้งสามเอนด์พอยต์และ webhook ทั้งสองตัว — ในรูปแบบ OpenAPI 3.1 เอกสารนี้สร้างจากสคีมาชุดเดียวกับที่ API ใช้ตรวจสอบ จึงเป็นไปไม่ได้ที่จะอธิบายเซิร์ฟเวอร์คนละตัว ใช้สร้างไคลเอนต์สำหรับภาษาที่ยังไม่มี SDK ได้

การสร้างใบแจ้งหนี้มีขีดจำกัดอัตราต่อโปรเจกต์: ของจริง 3,000/นาที และ 100,000/วัน ทดสอบ 300/นาที และ 10,000/วัน

## สร้างใบแจ้งหนี้

### `POST /v1/invoices`

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| ฟิลด์ | หมายเหตุ |
| --- | --- |
| `amount` | จำเป็น สตริงทศนิยมในช่วง `0.01`–`1000000.00` ทศนิยมได้ไม่เกิน 2 ตำแหน่ง จะถูกปรับรูปในผลลัพธ์ (`12.34` → `12.3400`) สกุลเงินถูกกำหนดตายตัวเป็น `USD` และคืนกลับมาในผลลัพธ์ ไม่ใช่ฟิลด์ของ request |
| `reference_id` | เลขอ้างอิงฝั่งคุณ ไม่บังคับ ต้องไม่ซ้ำภายในโปรเจกต์และโหมดเดียวกัน ยาวไม่เกิน 200 อักขระ ลองใหม่ด้วยเงื่อนไขเดิมจะได้ใบเดิมกลับมาพร้อม `200 OK` และ `meta.result: "reused"` ส่วนเงื่อนไขที่ต่างไปจะได้ `409 reference_id_conflict` |
| `description` | ข้อความที่ผู้จ่ายเห็น ไม่บังคับ ยาวไม่เกิน 500 อักขระ |
| `return_url` | URL `http(s)` ไม่บังคับ ยาวไม่เกิน 1000 อักขระ — ปุ่มกลับไปหาร้านค้าที่แสดงหลังจ่ายเงินเสร็จ ถ้าไม่ส่งมา ระบบจะคัดลอกค่าเริ่มต้นของโปรเจกต์ไว้ในใบแจ้งหนี้ ถ้าส่ง `null` หรือ `""` จะไม่มี URL กลับ เวลาลองใหม่ด้วย `reference_id` การไม่ส่ง `return_url` จะไม่ถูกนำไปเทียบกับใบเดิม ดังนั้นถ้าการลองใหม่ต้องยืนยันค่าใดค่าหนึ่ง ให้ส่งมาให้ชัดเจน |

`201 Created` หรือ `200 OK` เมื่อเป็นการใช้ซ้ำแบบ idempotent:

> SDK อย่างเป็นทางการคืนเฉพาะตัวทรัพยากรและตัด `meta.result` ทิ้ง ดังนั้นเมื่อผ่าน SDK การสร้างใหม่กับการใช้ซ้ำแบบ idempotent จะแยกไม่ออก ซึ่งเป็นจุดประสงค์ของ `reference_id` อยู่แล้ว ให้ยึดบัญชีฝั่งคุณกับ `reference_id` ที่คุณส่งไป และเรียกเอนด์พอยต์ตรง ๆ เมื่อต้องการแยกให้ได้

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

- **เงินไม่มีทางถูกเปลี่ยนเส้นทางผ่าน API นี้** request กำหนดที่อยู่ผู้รับหรือการตั้งค่า contract ไม่ได้ — ค่าเหล่านั้นมาจากการตั้งค่าโปรเจกต์ที่ยืนยันแล้วของคุณ หลังสร้างแล้ว ใบแจ้งหนี้จะเปลี่ยนแปลงไม่ได้อีก นอกจากสถานะการชำระเงินและการนำเงินเข้ากระเป๋า
- ใบแจ้งหนี้ทดสอบจะคืน `monitoring_ends_at: null`, `payment_options: []` และ `checkout_status: "unavailable"` ให้จ่ายผ่าน[ปลายทาง test-payments](#post-v1invoicesidtest-payments)

รหัสข้อผิดพลาด: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (รหัสฟิลด์ `invalid_format` หรือ `amount_too_small` / `amount_too_large` พร้อม `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`

`409 no_payment_options_available` หมายความว่าออกตัวเลือกการจ่ายเงินให้ใบนี้ไม่ได้เลย และจะมี `meta.reason_codes` ที่เรียงแล้วแนบมาด้วย ได้แก่ `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` โดย `merchant_address_provisioning` เป็นเรื่องชั่วคราว — ที่อยู่ Solana หรือ TRON ที่เพิ่งเพิ่มยังเปิดใช้งานไม่เสร็จ และ request เดิมมักสำเร็จภายในไม่กี่วินาที

## อ่านข้อมูลใบแจ้งหนี้

### `GET /v1/invoices/{id}`

มุมมองใบแจ้งหนี้ฝั่งผู้จ่าย: ข้อมูลสรุป สถานะการชำระเงิน แบรนด์ของโปรเจกต์ ข้อมูลการชำระเงิน และรายการโอนที่รับมาแล้ว ไม่ต้องใช้คีย์ API — นี่คือปลายทางที่หน้าชำระเงินเรียกซ้ำ ๆ

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

เทียบกับผลลัพธ์ตอนสร้าง ตรงนี้เพิ่ม `amount_paid`, `project` และ `transfers` เข้ามา แล้วตัด `reference_id` ที่เป็นของคุณคนเดียวออกไป `description`, `return_url`, `project.name` และ `project.logo_url` เป็น `null` ได้ทั้งหมด — หน้าชำระเงินต้องแสดงผลได้แม้ไม่มีสักอย่าง

`transfers` คือรายการหลักฐานการรับเงิน เพื่อให้หน้าชำระเงินแสดงธุรกรรมบนเชนแต่ละรายการและลิงก์ไปยัง block explorer ได้ แต่ละรายการมี `chain_namespace`, `chain_reference`, `transaction_id`, `event_index`, `amount` (ในสกุลเงินของใบแจ้งหนี้ ที่สเกลทศนิยม 18 ตำแหน่งเท่ากับ `amount_paid` และสำหรับ `direct_exact` จะไม่รวมส่วนเพิ่มที่ใช้จับคู่) และ `explorer_transaction_url` (เป็น `null` เมื่อไม่ได้ตั้งค่า explorer ไว้)

จะปรากฏเฉพาะการโอนที่ยืนยันแล้ว — รายการที่ยังรออยู่อาจหลุดไปตอนเชนจัดเรียงใหม่ — และจำกัดไว้ที่ 20 รายการที่ยอดสูงสุด เรียงจากมากไปน้อย เพื่อไม่ให้เศษเงินมาเบียดการจ่ายจริงตกไป ฟิลด์นี้มีอยู่เสมอ: เป็น `[]` จนกว่าจะมีการโอนที่ยืนยันแล้ว และเป็น `[]` เสมอสำหรับใบแจ้งหนี้ทดสอบ

ทั้ง id ที่รูปแบบไม่ถูกต้องและ id ที่ไม่มีอยู่จริงจะได้ `404 invoice_not_found` เหมือนกัน

## ผู้ซื้อจ่ายเงินอย่างไร

`payment_options` คือรายการช่องทางที่จ่ายใบแจ้งหนี้นี้ได้ หนึ่งรายการต่อเครือข่ายและโทเคน เรียงตามลำดับที่แสดงต่อผู้จ่าย: USDT มาก่อน USDC จากนั้นตามลำดับเครือข่ายของแต่ละโทเคน ตัวเลือกทั้งหมด รวมถึงที่อยู่และยอดเงินในนั้น ถูกล็อกไว้ตอนสร้างใบแจ้งหนี้ — ที่อยู่รับเงินหรือช่องทางที่คุณตั้งค่าทีหลังจะไม่ไปเขียนทับใบแจ้งหนี้ที่มีอยู่แล้ว มีเพียง `status` ของแต่ละตัวเลือกเท่านั้นที่คำนวณใหม่ทุกครั้งที่ตอบกลับ

ให้ระบุตัวเลือกด้วย (`chain_namespace`, `chain_reference`, `token_address`) อย่าอิงตำแหน่งในอาร์เรย์ และอย่าอิง `network_label`, `display_symbol`, `logo_url` หรือ `chain_logo_url` เพราะทั้งหมดนั้นเป็นข้อมูลสำหรับแสดงผลเท่านั้น

`collection_method` คือตัวจำแนก

**`evm_deposit`** — ที่อยู่รับเงินที่เป็นของใบแจ้งหนี้นี้ใบเดียว:

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

การโอนยอดบวกใด ๆ ที่มาทันเวลาจะนับเป็นยอดชำระตามจำนวนนั้น ส่วน `suggested_amount` เป็นเพียงคำแนะนำ ไม่ใช่เงื่อนไขว่ายอดต้องตรง: ค่านี้คือ `max(0, amount_due − pending)` ที่ปัด**ขึ้น**ตาม `token_decimals` หรือปัดที่ 6 ตำแหน่งเมื่อช่องทางนั้นมีมากกว่า เพราะคนต้องพิมพ์ตัวเลขชุดนี้เอง แล้วเติมศูนย์กลับให้ครบ `token_decimals` จึงอาจมากกว่า `amount_due` ได้ไม่เกิน `0.000001` อย่าถือว่าสองค่านี้เท่ากัน

**`direct_exact`** — ที่อยู่ Solana หรือ TRON ของคุณ พร้อมยอดเงินแบบเป๊ะ:

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

ผู้ซื้อต้องโอน `exact_amount` (`invoice_amount + matching_increment`) พอดีในการโอนครั้งเดียว ส่วนเพิ่มนี้คือกลไกที่ใช้ระบุว่าการจ่ายเป็นของใบไหน เงินก้อนนั้นถึงมือคุณ แต่ไม่มีวันถูกนับเป็นยอดชำระของใบแจ้งหนี้

เพราะยอดเป๊ะนั้นครอบคลุมใบแจ้งหนี้ทั้งใบเสมอ ตัวเลือก `direct_exact` จะกลายเป็น `unavailable` ทันทีที่ใบแจ้งหนี้มีการจ่ายที่ยืนยันแล้วหรือที่ยังรออยู่ ไม่ว่าจะมาจากช่องทางอื่นก็ตาม ด้วยเหตุผลเดียวกัน ใบแจ้งหนี้แบบโอนตรงที่จ่ายมาบางส่วนแล้วจะจ่ายเพิ่มให้ครบไม่ได้ ให้ออกใบแจ้งหนี้ใบใหม่สำหรับยอดที่เหลือแทน

เฉพาะตัวเลือกที่ `status: "ready"` เท่านั้นที่มีฟิลด์สำหรับจ่ายเงินตามข้างบน ส่วนตัวเลือก `unavailable` มีเฉพาะฟิลด์พื้นฐาน เพราะมันหยุดให้บริการไปแล้ว (อยู่ระหว่างตรวจสอบด้วยคน ที่อยู่หรือช่องทางถูกบล็อก เชนถูกพักการทำงาน หมดช่วงเวลาชำระเงิน) และไม่ควรแสดงให้ผู้ซื้อเห็นอีก

## สถานะการชำระเงิน

มีสองฟิลด์สถานะ และทั้งสองตอบคนละคำถาม

**`status`** คือสถานะทางบัญชีหลัก อ้างอิงจากการจ่ายที่ยืนยันแล้วและการนำเงินเข้ากระเป๋า ได้แก่ `unpaid`, `partially_paid`, `paid`, `settling`, `settled` หรือ `review_required` โดย `paid`, `settling` และ `settled` ล้วนหมายความว่าผู้ซื้อจ่ายแล้ว ต่างกันแค่ว่าเงินเดินทางไปถึงกระเป๋าของคุณแค่ไหน ส่วน `review_required` หมายถึงใบแจ้งหนี้กำลังรอการตรวจสอบด้วยคน — นี่**ไม่ใช่**สถานะจ่ายแล้ว อย่าจัดการคำสั่งซื้อแม้ `amount_paid` จะดูเพียงพอ

**`checkout_status`** คือสถานะในมุมของผู้จ่าย คำนวณใหม่ทุกการตอบกลับ และพิจารณาตามลำดับนี้:

| ค่า | ความหมาย |
| --- | --- |
| `paid` | `status` เป็น `paid`, `settling` หรือ `settled` |
| `confirming` | มีหลักฐานบนเชนมาแล้ว แต่ยังไม่ยืนยัน |
| `expired` | เลย `monitoring_ends_at` แล้ว |
| `open` | มีตัวเลือกการจ่ายที่ `ready` อย่างน้อยหนึ่งตัว |
| `unavailable` | กรณีที่เหลือทั้งหมด: อยู่ระหว่างตรวจสอบ ช่องทางถูกบล็อก หรือใบทดสอบที่ยังไม่จ่าย |

**`checkout_status` ไม่ใช่สัญญาณอนุญาตให้จัดการคำสั่งซื้อ** ให้ใช้ webhook `invoice.paid`

**`payment_revision`** เริ่มที่ `0` และเพิ่มขึ้นหนึ่งครั้งทุกครั้งที่ชุดการชำระเงินที่ยืนยันและนับยอดแล้วเปลี่ยนไป ได้แก่ มีการโอนใหม่ มีการกลับรายการ มีการจ่ายทดสอบใหม่ หรือมีการแก้เวลาบนเชนของการโอนที่นับยอดไปแล้ว ส่วนการนำเงินเข้ากระเป๋าอย่างเดียวไม่ทำให้ค่านี้ขยับ และค่านี้เปลี่ยนได้แม้ `status` ไม่เปลี่ยน ใช้ค่านี้เพื่อทิ้ง snapshot ของใบแจ้งหนี้หรือ webhook ที่มาถึงหลังของที่ใหม่กว่า

`amount_due` คือ `max(amount − amount_paid, 0)` และ `amount_overpaid` คือ `max(amount_paid − amount, 0)` ให้อ่านจากสองฟิลด์นี้ แทนที่จะลบยอดเงินเอง

## ช่วงเวลาชำระเงิน

`monitoring_ends_at` คือหนึ่งวันหลังจากสร้างใบแจ้งหนี้ และเป็นเส้นแบ่งเพียงเส้นเดียว การโอนจะถูกนับยอดโดยอัตโนมัติก็ต่อเมื่อเวลาบนเชนของการโอนนั้นอยู่ในช่วงเวลาดังกล่าว — ไม่นับอะไรที่เกิดก่อนใบแจ้งหนี้จะมีอยู่ และไม่นับอะไรที่มาตรง `monitoring_ends_at` พอดีหรือหลังจากนั้น นาฬิกาของคุณ เวลาที่เราสังเกตเห็น และเวลาที่ webhook มาถึง ไม่มีส่วนเกี่ยวข้องเลย

การจ่ายที่มาช้าไม่ได้หายไปไหน มันยังถูกบันทึกไว้กับใบแจ้งหนี้และแสดงในแดชบอร์ด ซึ่งคุณสามารถนับยอดให้มันได้โดยระบุธุรกรรมนั้นเอง — คำยืนยันแบบนี้ไม่มีกระบวนการอัตโนมัติไหนทำแทนคุณได้ ส่วนจะมีเวลานานแค่ไหนขึ้นอยู่กับช่องทาง:

- **EVM** — ไม่มีกำหนด ที่อยู่รับเงินเป็นของใบแจ้งหนี้ใบนี้ใบเดียวและไม่เคยถูกออกซ้ำ การโอนที่มาถึงที่อยู่นั้นจึงเป็นของใบอื่นไปไม่ได้
- **Solana และ TRON** — 21 วันนับจากวันสร้าง ยอดเป๊ะนั้นจะถูกกันไว้อีก 20 วันหลัง `monitoring_ends_at` หลังจากนั้นยอดดังกล่าวอาจเป็นของใบแจ้งหนี้ใบใหม่กว่าไปแล้ว และไม่มีใครฟันธงได้ว่าการโอนที่มาช้าจ่ายให้ใบไหน สิ่งที่นับคือเวลาที่การโอนมาถึง ไม่ใช่เวลาที่คุณมาจัดการมัน

ผลกระทบหนึ่งต่อการเชื่อมต่อของคุณ: **`invoice.paid` อาจมาถึงหลังจากหน้าชำระเงินขึ้น `expired` ไปนานแล้ว** และไม่มีฟิลด์ไหนแยกกรณีนี้ออกได้ ถ้าคุณยกเลิกหรือขายสินค้าซ้ำเมื่อหน้าชำระเงินหมดอายุ ให้เทียบ `invoice.paid` กับสถานะคำสั่งซื้อฝั่งคุณเอง แทนที่จะเหมาว่าใบแจ้งหนี้ยังเปิดอยู่ — และไม่ว่ากรณีไหนก็ให้ประมวลผลแบบ idempotent

## Webhook

ตั้งค่า URL ของ webhook ในแดชบอร์ด โหมดทดสอบและโหมดจริงต่างมี URL และคีย์สำหรับเซ็นของตัวเอง

### เหตุการณ์

**`invoice.paid`** — ใบแจ้งหนี้เข้าสู่สถานะจ่ายแล้ว (`paid`, `settling` หรือ `settled`):

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

**`invoice.payment_reversed`** — ใบแจ้งหนี้ที่เคยจ่ายครบแล้วตกกลับมาต่ำกว่ายอด เช่น การจัดเรียงเชนใหม่ทำให้การโอนที่เคยนับยอดไว้หายไป payload รูปแบบเดียวกัน พร้อม `status` ปัจจุบันของใบแจ้งหนี้ `payment_revision` ที่สูงขึ้น และ `fully_paid_at: null` ให้ถือว่าสัญญาณจ่ายเงินก่อนหน้าใช้ไม่ได้แล้ว โดยจัดการตามนโยบายธุรกิจของคุณเอง

- `review_required` ไม่มีวันทำให้เกิด `invoice.paid` ถ้ายอดข้ามเกณฑ์ระหว่างการตรวจสอบ เหตุการณ์จะถูกส่งครั้งเดียวหลังตรวจสอบเสร็จ
- ลำดับจริงที่เป็น จ่ายแล้ว → กลับรายการ → จ่ายแล้ว จะส่ง `invoice.paid` ตามด้วย `invoice.payment_reversed` แล้วตามด้วย `invoice.paid` ใหม่ แต่ละครั้งพร้อม `payment_revision` ที่ตรงกัน
- `reference_id` และ `fully_paid_at` เป็น `null` ได้ แต่มีอยู่เสมอ ส่วน `return_url` และข้อมูลการชำระเงินตั้งใจไม่ใส่มา — ให้กระทบยอดฝั่งเซิร์ฟเวอร์ด้วย id ของใบแจ้งหนี้บวกกับ `reference_id`

### การตรวจสอบลายเซ็น

ทุกครั้งที่ส่งจะมี:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` คือ Unix timestamp หน่วยวินาที ส่วน `v1` คือ HMAC-SHA256 ในรูปเลขฐานสิบหกตัวพิมพ์เล็กของ `<t>.<raw_body>` โดยใช้คีย์ webhook ของโหมดนั้น ให้ตรวจสอบกับ **body ดิบ** ก่อนแปลงเป็น JSON — การ serialize JSON ใหม่อาจทำให้ไบต์เปลี่ยนและลายเซ็นใช้ไม่ได้ และให้ปฏิเสธ timestamp ที่อยู่นอกช่วงที่คุณยอมรับได้ SDK ทางการทุกตัวมีฟังก์ชันช่วย `verifyWebhook` มาให้แล้ว

### การส่งและการส่งซ้ำ

- การส่งแต่ละครั้งหมดเวลาที่ 10 วินาที
- การตอบกลับที่ไม่ใช่ 2xx ทุกกรณี — **รวมถึง redirect และ 4xx** — บวกกับข้อผิดพลาดเครือข่ายและ timeout จะถูกส่งซ้ำโดยเว้นระยะแบบมีขอบเขต: 1 นาที, 5 นาที, 30 นาที แล้ว 2 ชั่วโมง แต่ละครั้งบวกลบสุ่มได้ถึง 20% รวมทั้งหมดห้าครั้ง
- การส่งเป็นแบบ **อย่างน้อยหนึ่งครั้ง และอาจมาไม่เรียงลำดับ** ให้กรองซ้ำด้วย `id` ของเหตุการณ์ เก็บ snapshot ที่ `payment_revision` สูงสุดไว้ และตอบ `2xx` ให้เร็ว — ค่อยไปทำงานต่อหลังตอบรับแล้ว

## โหมดทดสอบและการขึ้นใช้งานจริง

### `POST /v1/invoices/{id}/test-payments`

เพิ่มการจ่ายเงินจำลองให้ใบแจ้งหนี้**ทดสอบ** แล้วคืนสถานะการชำระเงินล่าสุด ใช้ได้กับคีย์ `sk_test_...` เท่านั้น — นี่คือวิธีไล่ครบวงจร ยังไม่จ่าย → จ่ายแล้ว → webhook โดยไม่ต้องแตะเชนเลย

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` จำเป็นและต้องมากกว่าศูนย์ จำนวนเต็มได้ไม่เกิน 15 หลักและทศนิยมไม่เกิน 4 ตำแหน่ง (`5`, `5.0` และ `5.0000` ปรับรูปเป็น `5.0000` เหมือนกันหมด)
- `reference_id` ไม่บังคับ ยาวไม่เกิน 200 อักขระ และเป็น idempotent ต่อใบแจ้งหนี้: เลขอ้างอิงเดิมกับยอดที่ปรับรูปแล้วเท่ากันจะได้ `200 OK` และ `meta.result: "reused"` ส่วนยอดที่ต่างไปจะได้ `409 test_payment_reference_conflict`
- จ่ายบางส่วน จ่ายครบ และจ่ายเกิน ทำได้หมด: เป็น `partially_paid` เมื่อ `0 < amount_paid < amount` และเป็น `paid` เมื่อ `amount_paid >= amount` การข้ามเข้าสู่ `paid` ครั้งแรกจะส่ง `invoice.paid` หนึ่งครั้ง และการจ่ายใหม่ทุกครั้งจะทำให้ `payment_revision` เพิ่มขึ้น
- จำกัดที่ 300 ครั้ง/นาที และ 10,000 ครั้ง/วัน ต่อโปรเจกต์

ผลลัพธ์คือใบแจ้งหนี้ในรูปแบบเดียวกับตอนสร้าง บวกกับ `amount_paid` และ `fully_paid_at` พร้อม `meta.result`

รหัสข้อผิดพลาด: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`

### ขึ้นใช้งานจริง

เมื่อวงจรทำงานกับ webhook ทดสอบของคุณได้แล้ว — พัฒนาบนเครื่องใช้ tunnel อย่าง ngrok หรือ cloudflared ได้:

1. สร้างคีย์ `sk_live_` ในแดชบอร์ด
2. ตั้ง URL webhook จริงของคุณ
3. สลับคีย์ในคอนฟิกเซิร์ฟเวอร์ของคุณ

อย่างอื่นไม่เปลี่ยนเลย: ปลายทางเดิม รูปแบบเดิม เพียงแต่ตอนนี้ใบแจ้งหนี้จริงมาพร้อม `payment_options` ของจริง ใบแจ้งหนี้ทดสอบและการจ่ายทดสอบไม่มีวันแตะเชน และไม่มีวันถูกนับเป็นการจ่ายจริง

## ความช่วยเหลือ

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- อีเมล: help@invoq.money
