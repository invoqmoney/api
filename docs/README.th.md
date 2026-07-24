# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · **ไทย** · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> เอกสารนี้แปลจาก README ภาษาอังกฤษ หากมีข้อความไม่ตรงกัน ให้ยึด[ฉบับภาษาอังกฤษ](../README.md)เป็นหลัก

ระบบรับชำระเงินด้วย stablecoin สำหรับนักพัฒนาอิสระ แบบ non-custodial — เงินเข้ากระเป๋าเงินของคุณเองโดยตรง

นี่คือเอกสารอ้างอิงของ REST API สาธารณะของ invoq ถ้าคุณใช้ SDK ทางการตัวใดตัวหนึ่ง ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)) SDK เหล่านั้นห่อปลายทางชุดเดียวกันนี้ทุกประการ — เอกสารนี้คือสัญญาที่ SDK ทุกตัวยึดตาม

- **Base URL:** `https://api.invoq.money`
- **หน้าชำระเงินที่โฮสต์ให้:** `https://pay.invoq.money/<invoice id>`
- **แดชบอร์ด** (คีย์ API, กระเป๋าเงินสำหรับรับเงิน, webhook): `https://app.invoq.money`

## หลักการทำงาน

1. **สร้างใบแจ้งหนี้**จากเซิร์ฟเวอร์ของคุณ (`POST /v1/invoices`)
2. **ให้ผู้ซื้อจ่าย** วิธีที่ง่ายที่สุด: ส่งผู้ซื้อไปที่หน้าชำระเงินที่โฮสต์ให้ที่ `https://pay.invoq.money/<invoice id>` — หน้านี้แสดงยอดเงิน ที่อยู่รับเงิน และ QR code รองรับสิบภาษา และคุณไม่ต้องทำ UI เองเลยแม้แต่น้อย หรือจะฝังหน้าชำระเงินเดียวกันนี้ในเว็บไซต์ของคุณเองด้วย [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) ก็ได้ ผู้ซื้อโอน USDT หรือ USDC จากกระเป๋าเงินหรือกระดานเทรดไหนก็ได้
3. **รับแจ้งเมื่อจ่ายแล้ว** invoq ยืนยันการโอนบนเชน แล้วส่ง webhook `invoice.paid` ไปยังเซิร์ฟเวอร์ของคุณ ส่วนเงินเข้ากระเป๋าเงินของคุณเองโดยตรง

## เริ่มต้นอย่างรวดเร็ว

หยิบคีย์ทดสอบ (`sk_test_...`) จากแดชบอร์ด แล้วสร้างใบแจ้งหนี้ใบแรกของคุณ:

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

สำหรับใบแจ้งหนี้จริง แค่ `id` จากการตอบกลับก็เพียงพอแล้ว — `https://pay.invoq.money/<id>` คือหน้าชำระเงินของคุณ แต่ใบนี้เป็นใบแจ้งหนี้ทดสอบซึ่งไม่มีรายละเอียดการชำระเงินบนเชน จึงให้จำลองการจ่ายแทน:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

ถ้าคุณตั้ง URL webhook ทดสอบไว้ในแดชบอร์ด ปลายทางนั้นจะได้รับ `invoice.paid` ที่ลงลายเซ็นแล้ว เท่านี้ก็ครบทั้งวงจร — ที่เหลือของเอกสารนี้คือรายละเอียด

## การยืนยันตัวตน

ปลายทางฝั่งเซิร์ฟเวอร์ใช้คีย์ลับ (secret key) ที่สร้างในแดชบอร์ด:

```http
Authorization: Bearer sk_test_...
```

- คีย์ `sk_test_...` สร้าง**ใบแจ้งหนี้ทดสอบ**: การจ่ายเป็นการจำลอง ไม่มีอะไรเคลื่อนไหวบนเชน แต่ webhook เป็นของจริง (ลงลายเซ็นและส่งเหมือนโหมดจริงทุกอย่าง)
- คีย์ `sk_live_...` สร้าง**ใบแจ้งหนี้จริง**พร้อมรายละเอียดการชำระเงินบนเชนของจริง

โหมดของใบแจ้งหนี้มาจากคีย์เสมอ — เนื้อหา request ไม่รับฟิลด์ `mode` ไม่ว่ากรณีใด เก็บคีย์ลับไว้บนเซิร์ฟเวอร์ของคุณ อย่าใส่ลงในโค้ดฝั่งไคลเอนต์เด็ดขาด

## รูปแบบการตอบกลับ

การตอบกลับที่สำเร็จจะห่อทรัพยากรไว้ใน `data`:

```json
{ "data": { "id": "inv_..." } }
```

การตอบกลับข้อผิดพลาดใช้รูปแบบเดียวกันทุกปลายทาง:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` คือรหัสข้อผิดพลาดที่คงที่และอ่านได้ด้วยโปรแกรม — ให้แยกกรณีจากค่านี้ ไม่ใช่จาก `message`
- `fields` จะมีเฉพาะกรณีข้อผิดพลาดจากการตรวจสอบระดับฟิลด์เท่านั้น
- บริบททางธุรกิจเพิ่มเติมจะส่งกลับใน `meta` เช่น `retry_after`, `reason_codes` หรือ `min_amount`

เนื้อหา request จำกัดขนาดที่ 4KB ถ้าเกินจะได้ `413 request_body_too_large` กลับไป

## สร้างใบแจ้งหนี้

### `POST /v1/invoices`

สร้างใบแจ้งหนี้ แล้วคืนข้อมูลสรุปพร้อมรายละเอียดการชำระเงิน

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
| `amount` | จำเป็นต้องระบุ สตริงเลขทศนิยม `0.01`–`1000000.00` ทศนิยมไม่เกิน 2 ตำแหน่ง ในการตอบกลับจะถูกปรับรูปแบบ (`12.34` → `12.3400`) สกุลเงินทางธุรกิจถูกกำหนดตายตัวเป็น `USD` และส่งกลับมาในการตอบกลับ ไม่ใช่ฟิลด์ใน request |
| `reference_id` | ระบุหรือไม่ก็ได้ ใช้อ้างอิงฝั่งผู้เรียก ต้องไม่ซ้ำต่อโปรเจกต์ + โหมด สูงสุด 200 อักขระ ลองใหม่ด้วยเงื่อนไขเดิมทุกอย่างจะได้ใบแจ้งหนี้ใบเดิมกลับมาพร้อม `200 OK` ส่วนเงื่อนไขที่ต่างกันจะได้ `409 reference_id_conflict` |
| `description` | ระบุหรือไม่ก็ได้ ข้อความที่แสดงให้ผู้จ่ายเห็น สูงสุด 500 อักขระ |
| `return_url` | ระบุหรือไม่ก็ได้ URL แบบ `http(s)` ที่จะแสดงเป็นปุ่มกลับไปยังร้านค้าหลังชำระเงิน สูงสุด 1000 อักขระ ไม่ส่งมา → ระบบบันทึกค่า return URL เริ่มต้นของโปรเจกต์ ณ ตอนนั้นติดใบแจ้งหนี้ไว้ ส่ง `null` หรือ `""` มาโดยตรง → ไม่มี return URL ตอนลองใหม่ด้วย `reference_id` เดิม `return_url` ที่ไม่ได้ส่งมาจะไม่ถูกตรวจเทียบกับใบแจ้งหนี้เดิม — ถ้าการลองใหม่ต้องยืนยันค่าใดค่าหนึ่งโดยเฉพาะ ให้ส่งมาอย่างชัดเจน |

การตอบกลับเมื่อสำเร็จ (`201 Created` ส่วนการใช้ซ้ำแบบ idempotent จะได้ `200 OK` พร้อม `meta.result: "reused"`):

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

พฤติกรรมที่ควรรู้:

- **เงินไม่มีทางถูกเปลี่ยนเส้นทางผ่าน API นี้** request ไม่สามารถกำหนดที่อยู่ผู้รับหรือการตั้งค่า contract ได้ — ค่าเหล่านั้นมาจากการตั้งค่าโปรเจกต์ที่ยืนยันแล้วของคุณ ณ ตอนสร้างใบแจ้งหนี้ หลังสร้างแล้ว ใบแจ้งหนี้จะเปลี่ยนแปลงไม่ได้อีก นอกจากสถานะการชำระเงินและการนำเงินเข้ากระเป๋า
- **`payment_options` คือรายการช่องทางที่ผู้ซื้อจ่ายได้** หนึ่งรายการต่อเครือข่าย + โทเคน เรียงตามลำดับที่แสดงต่อผู้จ่าย (USDT มาก่อน USDC จากนั้นตามลำดับเครือข่ายที่ผ่านการพิจารณาแล้วของแต่ละโทเคน) โดยมี `collection_method` เป็นตัวจำแนก:
  - `evm_deposit` — ที่อยู่รับเงินแบบ EVM แยกต่อใบแจ้งหนี้ การโอนยอดบวกใด ๆ ที่มาทันเวลาจะนับเป็นยอดชำระตามจำนวนนั้น ส่วน `suggested_amount` (`max(0, amount_due − pending)`) เป็นเพียงคำแนะนำ ไม่ใช่เงื่อนไขว่ายอดต้องตรง
  - `direct_exact` — ที่อยู่ร้านค้าบน Solana/TRON พร้อมยอดเงินแบบเป๊ะ ผู้ซื้อต้องโอน `exact_amount` (`invoice_amount + matching_increment`) พอดีในการโอนครั้งเดียว ส่วนเพิ่มนี้คือกลไกที่ใช้ระบุว่าการจ่ายเป็นของใบไหน และไม่มีวันถูกนับเป็นยอดชำระของใบแจ้งหนี้
  - เฉพาะตัวเลือกที่ `status: "ready"` เท่านั้นที่มีฟิลด์สำหรับจ่ายเงินตามข้างบน ส่วนตัวเลือก `unavailable` มีเฉพาะฟิลด์พื้นฐาน ให้ระบุตัวเลือกด้วย (`chain_namespace`, `chain_reference`, `token_address`) เท่านั้น อย่าอิงตำแหน่งในอาร์เรย์หรือข้อมูลที่ใช้แสดงผล
- **`checkout_status` คือสถานะในมุมของผู้จ่าย** คำนวณใหม่ทุกการตอบกลับ: `paid` (สถานะหลักเป็น paid/settling/settled), `confirming` (รอหลักฐานบนเชน), `expired` (เลย `monitoring_ends_at` แล้ว), `open` (มีตัวเลือกที่พร้อมอย่างน้อยหนึ่งตัว) หรือ `unavailable` ค่านี้ไม่ใช่สัญญาณอนุญาตให้จัดการคำสั่งซื้อ — ให้ใช้ webhook `invoice.paid`
- **`payment_revision`** เริ่มที่ `0` และเพิ่มขึ้นหนึ่งครั้งพอดีทุกครั้งที่ชุดการชำระเงินที่ยืนยันและนับยอดแล้วเปลี่ยนไป (การจ่ายทดสอบใหม่แต่ละครั้งก็ด้วย) ใช้ค่านี้เพื่อทิ้ง snapshot ของใบแจ้งหนี้หรือ webhook รุ่นเก่าที่มาถึงหลังของที่ใหม่กว่า
- **ยอดเงินเป็นสตริงเลขทศนิยมเสมอ ไม่มีวันเป็น float** ยอดที่จ่ายแล้ว/ค้างจ่ายใช้ทศนิยม 18 ตำแหน่ง `amount_due` คือ `max(amount − amount_paid, 0)` และ `amount_overpaid` คือ `max(amount_paid − amount, 0)` — ให้อ่านจากฟิลด์เหล่านี้แทนการลบเงินเอง
- **invoq เฝ้าดูเชนให้ 7 วัน**หลังสร้างใบแจ้งหนี้ (`monitoring_ends_at`) การโอนที่มาถึง ณ หรือหลังเวลานั้นจะถูกบันทึกไว้แต่ไม่นับเป็นยอดชำระ ส่วนการกระทบยอดด้วยตนเองในแดชบอร์ดคือเครื่องมือสำรองของผู้ดูแลสำหรับเคสพิเศษภายในช่วงเวลาเฝ้าดู
- ใบแจ้งหนี้ทดสอบคืน `monitoring_ends_at: null`, `payment_options: []` และ `checkout_status: "unavailable"` — การจ่ายใช้วิธีจำลองผ่านปลายทาง test-payments ด้านล่าง
- ลิมิตการเรียกต่อโปรเจกต์: โหมดจริง 3,000 ครั้ง/นาที และ 100,000 ครั้ง/วัน โหมดทดสอบ 300 ครั้ง/นาที และ 10,000 ครั้ง/วัน

รหัสข้อผิดพลาด: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (พร้อมรหัสระดับฟิลด์ `amount_too_small` / `amount_too_large` และ `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (พร้อม `meta.reason_codes` เรียงลำดับแล้ว: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — โดย `merchant_address_provisioning` เป็นสถานะชั่วคราว มักหายไปเองภายในหนึ่งถึงสองนาที), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`

## อ่านข้อมูลใบแจ้งหนี้

### `GET /v1/invoices/{id}`

คืนข้อมูลสรุปใบแจ้งหนี้แบบสาธารณะ สถานะการชำระเงินในมุมของผู้จ่าย ข้อมูลแบรนด์ของโปรเจกต์ และรายละเอียดการชำระเงิน **ไม่ต้องใช้คีย์ API** — id ของใบแจ้งหนี้เป็น id สาธารณะที่แชร์ได้และเดาไม่ได้ ใช้อยู่ใน URL ของลิงก์ชำระเงิน ปลายทางนี้จึงเป็นปลายทางที่ UI ชำระเงินใช้ poll (CORS อนุญาตทุก origin สำหรับ GET)

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

- `status` คือสถานะหลักของใบแจ้งหนี้ อิงจากเหตุการณ์การชำระเงินและการนำเงินเข้ากระเป๋าที่ยืนยันแล้ว: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` หรือ `review_required`
- `review_required` แปลว่าใบแจ้งหนี้กำลังรอการตรวจสอบโดยเจ้าหน้าที่ นี่**ไม่ใช่**สถานะชำระแล้ว — อย่าจัดการคำสั่งซื้อจากสถานะนี้ ต่อให้ `amount_paid` ดูครบแล้วก็ตาม
- `checkout_status` คือสถานะในมุมของผู้จ่ายที่คำนวณให้ตามที่อธิบายไว้ข้างต้น ใบแจ้งหนี้จริงที่มีหลักฐานบนเชนรอยืนยันจะแสดง `confirming` ขณะที่ `status` หลักยังคงเดิมจนกว่าการโอนจะยืนยันสำเร็จ
- `transfers` คือหลักฐานการรับเงินในมุมของผู้จ่าย: รายการโอนขาเข้าที่ยืนยันแล้วและนับเป็นยอดชำระของใบแจ้งหนี้ใบนี้ เพื่อให้หน้าชำระเงินแสดงธุรกรรมบนเชนแต่ละรายการพร้อมลิงก์ไป block explorer ได้ แต่ละรายการมี `chain_namespace`, `chain_reference`, `transaction_id` แบบมาตรฐาน, `event_index`, `amount` (หน่วยสกุลเงินของใบแจ้งหนี้ สเกลทศนิยม 18 ตำแหน่งเดียวกับ `amount_paid` โดยกรณี `direct_exact` จะไม่รวมส่วนเพิ่มสำหรับจับคู่) และ `explorer_transaction_url` (หรือ `null`) เฉพาะการโอนที่ยืนยันแล้วเท่านั้นที่แสดง — รายการที่ยังรอยืนยันอาจถูก reorg ลบทิ้งได้ — และจำกัดไว้ที่ 20 รายการที่ยอดสูงสุด เพื่อไม่ให้เศษเงิน (dust) ที่ถูกส่งเข้าที่อยู่รับเงินสาธารณะเบียดการจ่ายจริงออกไป ฟิลด์นี้มีอยู่เสมอ: เป็น `[]` จนกว่าจะมีการโอนที่ยืนยันแล้ว และเป็น `[]` เสมอสำหรับใบแจ้งหนี้ทดสอบ
- `reference_id` ซึ่งเป็นข้อมูลเฉพาะฝั่งผู้เรียกจะไม่ถูกใส่มาที่นี่ จะส่งกลับเฉพาะฟิลด์แบรนด์ `project` ในมุมของผู้จ่ายเท่านั้น
- id ที่รูปแบบไม่ถูกต้องและ id ที่ไม่รู้จัก ได้ `404 invoice_not_found` เหมือนกันทั้งคู่

## จำลองการจ่ายเงินทดสอบ

### `POST /v1/invoices/{id}/test-payments`

เพิ่มการจ่ายแบบจำลองให้ใบแจ้งหนี้**ทดสอบ** แล้วคืนสถานะการชำระเงินที่อัปเดตแล้ว ใช้ได้เฉพาะกับคีย์ `sk_test_...` เท่านั้น — นี่คือวิธีที่คุณทดสอบวงจร unpaid → paid → webhook ให้ครบทั้งเส้นโดยไม่ต้องแตะเชนเลย

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` จำเป็นต้องระบุ ต้องมากกว่าศูนย์ จำนวนเต็มไม่เกิน 15 หลักและทศนิยมไม่เกิน 4 ตำแหน่ง (`5`, `5.0`, `5.0000` ถูกปรับเป็น `5.0000` เหมือนกัน)
- `reference_id` ระบุหรือไม่ก็ได้ สูงสุด 200 อักขระ และ idempotent ต่อใบแจ้งหนี้: ใช้ซ้ำด้วยยอดหลังปรับรูปแบบที่เท่ากันจะได้ `200 OK` พร้อม `meta.result: "reused"` ส่วนยอดที่ต่างกันจะได้ `409 test_payment_reference_conflict`
- จ่ายบางส่วน จ่ายครบ หรือจ่ายเกินได้ทั้งนั้น: เป็น `partially_paid` ระหว่าง `0 < amount_paid < amount` และเป็น `paid` เมื่อ `amount_paid >= amount` การเปลี่ยนเข้าสู่ `paid` ครั้งแรกจะสร้าง webhook `invoice.paid` เชิงตรรกะหนึ่งรายการ และการจ่ายที่สร้างแต่ละรายการจะเพิ่มค่า `payment_revision`
- การสร้างจำกัดที่ 300 ครั้ง/นาที และ 10,000 ครั้ง/วัน ต่อโปรเจกต์

รหัสข้อผิดพลาด: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`

## Webhook

ตั้งค่า URL ของ webhook ในแดชบอร์ด — โหมดทดสอบและโหมดจริงต่างมี URL และซีเคร็ตสำหรับลงลายเซ็นของตัวเอง

### เหตุการณ์

**`invoice.paid`** — ส่งเมื่อใบแจ้งหนี้เปลี่ยนเข้าสู่สถานะชำระแล้ว (`paid`, `settling` หรือ `settled`):

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

**`invoice.payment_reversed`** — ส่งเมื่อใบแจ้งหนี้ที่เคยชำระแล้วมียอดตกกลับลงไปต่ำกว่าจำนวนของใบ (เช่น reorg บนเชนลบการโอนที่เคยนับยอดออกไป) รูปแบบ payload เหมือนกัน โดยมี `status` และ `amount_paid` ปัจจุบันของใบแจ้งหนี้ `payment_revision` ที่สูงขึ้น และ `fully_paid_at: null` ให้ถือว่าเหตุการณ์นี้ลบล้างสัญญาณจัดการคำสั่งซื้อก่อนหน้า ตามนโยบายธุรกิจของคุณเอง

- `review_required` ไม่มีวันทำให้เกิด `invoice.paid` เหตุการณ์จะถูกสร้างก็ต่อเมื่อการตรวจสอบผ่านและใบแจ้งหนี้เข้าสู่สถานะชำระแล้วเท่านั้น
- ลำดับจ่ายแล้ว → ย้อนกลับ → จ่ายแล้วที่เกิดขึ้นจริง จะส่ง `invoice.paid`, `invoice.payment_reversed` แล้วตามด้วย `invoice.paid` รายการใหม่ แต่ละเหตุการณ์มาพร้อม `payment_revision` ที่เป็นผลลัพธ์ของมันเอง
- `reference_id` และ `fully_paid_at` เป็น null ได้แต่มีมาเสมอ ส่วน `return_url` กับรายละเอียดการชำระเงินตั้งใจไม่ใส่มา ให้กระทบยอดฝั่งเซิร์ฟเวอร์ด้วย id ของใบแจ้งหนี้ควบคู่กับ `reference_id`

### การตรวจสอบลายเซ็น

webhook ขาออกทุกรายการมาพร้อม:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` คือ timestamp แบบ Unix เป็นวินาที ส่วน `v1` คือ HMAC-SHA256 ฐานสิบหกตัวพิมพ์เล็กของ `<t>.<raw_body>` โดยใช้ซีเคร็ต webhook ของโหมดนั้น ๆ เป็นคีย์ ให้ตรวจสอบกับ**เนื้อหา request ดิบ**ก่อน parse — การ serialize JSON ใหม่อาจทำให้ไบต์เปลี่ยนจนลายเซ็นใช้ไม่ได้ และให้ปฏิเสธ timestamp ที่อยู่นอกช่วงเวลากัน replay ที่คุณยอมรับ (SDK ทางการมี helper `verifyWebhook` ให้)

### การส่งและการส่งซ้ำ

- POST ที่ส่งไปจะหมดเวลารอที่ 10 วินาที
- ทุกการตอบกลับที่ไม่ใช่ 2xx — **รวมถึง redirect และ 4xx** — บวกกับข้อผิดพลาดเครือข่ายและ timeout จะถูกส่งซ้ำด้วย backoff แบบมีขอบเขต: 1 นาที, 5 นาที, 30 นาที แล้วตามด้วย 2 ชั่วโมง แต่ละช่วงมี jitter ได้ถึง 20% รวมความพยายามทั้งหมดสูงสุด 5 ครั้ง
- การส่งเป็นแบบ **at-least-once** และอาจมาไม่เรียงลำดับ: ให้กันซ้ำด้วย `id` ของเหตุการณ์ เก็บ snapshot ที่มี `data.invoice.payment_revision` มากที่สุด และตอบ `2xx` ให้เร็ว (ค่อยทำงานจริงหลังตอบรับแล้ว)

## ขึ้นใช้งานจริง

เมื่อวงจรในหัวข้อ[เริ่มต้นอย่างรวดเร็ว](#เริ่มต้นอย่างรวดเร็ว)ทำงานกับ webhook ทดสอบของคุณได้แล้ว (พัฒนาบนเครื่องใช้ tunnel อย่าง ngrok หรือ cloudflared ได้):

1. สร้างคีย์ `sk_live_` ในแดชบอร์ด
2. ตั้ง URL webhook จริงของคุณในแดชบอร์ด
3. สลับคีย์ในคอนฟิกเซิร์ฟเวอร์ของคุณ อย่างอื่นไม่เปลี่ยนเลย: ปลายทางเดิม รูปแบบเดิม — เพียงแต่ตอนนี้ใบแจ้งหนี้จริงมาพร้อม `payment_options` ของจริง

ใบแจ้งหนี้ทดสอบและการจ่ายทดสอบไม่มีวันแตะเชน และไม่มีวันถูกนับเป็นการจ่ายจริง

## ความช่วยเหลือ

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- อีเมล: help@invoq.money
