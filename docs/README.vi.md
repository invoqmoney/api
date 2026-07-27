# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · **Tiếng Việt** · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Tài liệu này được dịch từ README tiếng Anh; nếu có chỗ khác nhau, [bản tiếng Anh](../README.md) là bản chuẩn.

Thanh toán stablecoin cho lập trình viên độc lập. Không giữ hộ tiền — tiền về thẳng ví của bạn.

Đây là tài liệu tham chiếu cho REST API công khai của invoq. Các SDK chính thức — [Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby) — bọc đúng những endpoint này.

- **Base URL:** `https://api.invoq.money`
- **Trang thanh toán được lưu trữ sẵn:** `https://pay.invoq.money/<id hóa đơn>`
- **Bảng điều khiển** (khóa API, ví nhận tiền, webhook): `https://app.invoq.money`

**Đang code bằng AI? Dán câu này.**

```
Thêm thanh toán stablecoin vào dự án của tôi bằng invoq. Bắt đầu ở chế độ thử nghiệm. Đọc tài liệu trước khi viết code: https://invoq.money/llms.txt
```

## Cách hoạt động

1. **Tạo hóa đơn** từ máy chủ của bạn (`POST /v1/invoices`).
2. **Để người mua thanh toán.** Cách dễ nhất: đưa họ tới `https://pay.invoq.money/<id hóa đơn>` — trang này hiển thị số tiền, địa chỉ và mã QR, hỗ trợ mười ngôn ngữ, và bạn không phải viết một dòng giao diện nào. Hoặc nhúng chính trang đó bằng [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). Người mua gửi USDT hoặc USDC từ bất kỳ ví hay sàn nào.
3. **Được báo khi đã thanh toán.** invoq xác nhận giao dịch trên chuỗi rồi gửi webhook `invoice.paid` tới máy chủ của bạn. Tiền về thẳng ví của chính bạn.

## Bắt đầu nhanh

Lấy một khóa thử nghiệm (`sk_test_...`) trong bảng điều khiển rồi tạo hóa đơn đầu tiên:

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

Với hóa đơn chạy thật, `id` trong phản hồi là tất cả những gì bạn cần — `https://pay.invoq.money/<id>` chính là trang thanh toán của bạn. Hóa đơn trên là hóa đơn thử nghiệm, không kèm hướng dẫn thanh toán trên chuỗi, nên hãy mô phỏng khoản thanh toán:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Nếu bạn đã đặt URL webhook thử nghiệm trong bảng điều khiển, nó sẽ nhận một `invoice.paid` có chữ ký. Đó là toàn bộ vòng lặp — phần còn lại của tài liệu này là chi tiết.

## Xác thực

Các endpoint phía máy chủ dùng khóa API bí mật tạo trong bảng điều khiển:

```http
Authorization: Bearer sk_test_...
```

- Khóa `sk_test_...` tạo **hóa đơn thử nghiệm**: thanh toán được mô phỏng, không có gì chuyển động trên chuỗi, còn webhook là thật — được ký và gửi y như khi chạy thật.
- Khóa `sk_live_...` tạo **hóa đơn thật** kèm hướng dẫn thanh toán trên chuỗi thật.

Chế độ luôn lấy từ khóa; phần thân request không bao giờ nhận trường `mode`. Hãy giữ khóa bí mật ở máy chủ, đừng bao giờ đưa vào mã phía client.

`GET /v1/invoices/{id}` không cần khóa. Id hóa đơn là id công khai không thể đoán, dùng trong các liên kết thanh toán, nên giao diện thanh toán có thể gọi thẳng từ trình duyệt — CORS cho phép mọi origin với `GET` và `HEAD`.

## Request và phản hồi

Phản hồi thành công bọc tài nguyên trong `data`:

```json
{ "data": { "id": "inv_..." } }
```

Mọi endpoint dùng chung một cấu trúc lỗi:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- Hãy rẽ nhánh theo `code`. `message` dành cho người đọc và có thể thay đổi bất cứ lúc nào.
- `fields` chỉ xuất hiện với lỗi kiểm tra ở mức trường; `location` là `body`, `query` hoặc `path`.
- Thông tin nghiệp vụ bổ sung nằm trong `meta`: `retry_after`, `reason_codes`, `min_amount`, v.v.

Vài quy ước nên biết:

- **Việc kiểm tra rất nghiêm ngặt.** Một khóa lạ trong thân request hay một tham số truy vấn lạ đều trả về `400 invalid_request` kèm `fields[].code: "unknown_field"` — gắn thêm một tham số chống cache sẽ làm hỏng cả request.
- **Số tiền luôn là chuỗi thập phân, không bao giờ là số thực dấu phẩy động.** Số tiền hóa đơn có 4 chữ số thập phân, số đã trả và còn phải trả có 18, còn số tiền theo token trong `payment_options` đúng bằng `token_decimals`.
- **`429 rate_limited` đặt gợi ý ngay trong thân phản hồi**, ở `meta.retry_after` tính bằng giây tròn. Không có header `Retry-After` nào được gửi — hãy đọc `meta`.
- Thân request giới hạn 4KB; vượt quá sẽ là `413 request_body_too_large`.
- Mọi phản hồi JSON đều kèm `Cache-Control: no-store` — trạng thái thanh toán được hỏi liên tục, nên không nơi nào được trả cho bạn một hóa đơn cũ.
- `GET /` là endpoint kiểm tra sống không cần xác thực. Nó trả về `204 No Content`.

Việc tạo hóa đơn bị giới hạn tần suất theo dự án: chạy thật 3.000/phút và 100.000/ngày, thử nghiệm 300/phút và 10.000/ngày.

## Tạo hóa đơn

### `POST /v1/invoices`

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Trường | Ghi chú |
| --- | --- |
| `amount` | Bắt buộc. Chuỗi thập phân, `0.01`–`1000000.00`, tối đa 2 chữ số thập phân. Được chuẩn hóa trong phản hồi (`12.34` → `12.3400`). Đơn vị tiền cố định là `USD` và được trả về trong phản hồi; đây không phải trường của request. |
| `reference_id` | Mã tham chiếu tùy chọn của bạn, duy nhất theo dự án và chế độ, tối đa 200 ký tự. Thử lại với cùng điều khoản sẽ trả về hóa đơn đã có với `200 OK` và `meta.result: "reused"`; điều khoản khác đi sẽ trả `409 reference_id_conflict`. |
| `description` | Văn bản tùy chọn hiển thị cho người thanh toán, tối đa 500 ký tự. |
| `return_url` | URL `http(s)` tùy chọn, tối đa 1000 ký tự — nút quay lại cửa hàng hiển thị sau khi thanh toán. Nếu bỏ trống, giá trị mặc định của dự án được chụp lại vào hóa đơn; truyền `null` hoặc `""` nếu không muốn có URL quay lại. Khi thử lại bằng `reference_id`, một `return_url` bị bỏ trống sẽ không được đối chiếu với hóa đơn đã có, nên hãy truyền tường minh khi lần thử lại đó cần khẳng định một giá trị. |

`201 Created`, hoặc `200 OK` khi dùng lại theo cơ chế idempotent:

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

- **Không thể đổi hướng dòng tiền qua API này.** Request không thể đặt địa chỉ nhận tiền hay cấu hình hợp đồng — chúng lấy từ cài đặt dự án đã xác minh của bạn. Sau khi tạo, hóa đơn là bất biến, chỉ trạng thái thanh toán và trạng thái tiền về ví là thay đổi.
- Hóa đơn thử nghiệm trả về `monitoring_ends_at: null`, `payment_options: []` và `checkout_status: "unavailable"`. Hãy thanh toán chúng qua [endpoint test-payments](#post-v1invoicesidtest-payments).

Mã lỗi: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (mã trường `invalid_format`, hoặc `amount_too_small` / `amount_too_large` kèm `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

`409 no_payment_options_available` nghĩa là không phát hành được tùy chọn thanh toán nào, và kèm `meta.reason_codes` đã sắp xếp: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted`. `merchant_address_provisioning` chỉ là tạm thời — một địa chỉ Solana hoặc TRON mới vẫn đang được kích hoạt, và cùng request đó thường thành công sau vài giây.

## Đọc hóa đơn

### `GET /v1/invoices/{id}`

Góc nhìn hóa đơn từ phía người thanh toán: tóm tắt, trạng thái thanh toán, thương hiệu của dự án, hướng dẫn thanh toán và danh sách giao dịch đã nhận. Không cần khóa API — đây chính là endpoint mà giao diện thanh toán gọi định kỳ.

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

So với phản hồi lúc tạo, ở đây có thêm `amount_paid`, `project` và `transfers`, đồng thời bỏ `reference_id` vốn chỉ dành cho bạn. `description`, `return_url`, `project.name` và `project.logo_url` đều có thể là `null` — trang thanh toán phải hiển thị được ngay cả khi thiếu tất cả.

`transfers` là danh sách biên nhận, để trang thanh toán hiển thị từng giao dịch trên chuỗi và liên kết tới trình khám phá khối. Mỗi mục có `chain_namespace`, `chain_reference`, `transaction_id`, `event_index`, `amount` (theo đơn vị tiền của hóa đơn, cùng thang 18 chữ số thập phân với `amount_paid`; với `direct_exact` thì không bao gồm phần cộng thêm để khớp) và `explorer_transaction_url` (bằng `null` khi không có trình khám phá nào được cấu hình).

Chỉ những giao dịch đã xác nhận mới xuất hiện — giao dịch đang chờ vẫn có thể bị mất khi chuỗi tổ chức lại — và giới hạn ở 20 khoản lớn nhất theo số tiền, xếp từ lớn xuống, để những khoản bụi không đẩy khoản thanh toán thật ra khỏi danh sách. Trường này luôn có mặt: là `[]` khi chưa giao dịch nào được xác nhận, và luôn `[]` với hóa đơn thử nghiệm.

Cả id sai định dạng lẫn id không tồn tại đều trả về `404 invoice_not_found`.

## Người mua trả tiền thế nào

`payment_options` liệt kê các cách trả cho hóa đơn này, mỗi mục ứng với một mạng và một token, theo thứ tự hiển thị cho người thanh toán: USDT trước USDC, rồi đến thứ tự mạng của từng token. Các tùy chọn, cùng địa chỉ và số tiền trong đó, được chốt lúc tạo hóa đơn — địa chỉ nhận tiền hay tuyến bạn cấu hình về sau không viết lại hóa đơn đã có. Chỉ `status` của từng tùy chọn là được tính lại ở mỗi phản hồi.

Hãy nhận diện một tùy chọn bằng bộ (`chain_namespace`, `chain_reference`, `token_address`). Đừng bao giờ dựa vào vị trí trong mảng, cũng đừng dựa vào `network_label`, `display_symbol`, `logo_url` hay `chain_logo_url` — đó chỉ là dữ liệu để hiển thị.

`collection_method` là trường phân biệt.

**`evm_deposit`** — một địa chỉ nạp tiền chỉ thuộc về hóa đơn này:

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

Mọi khoản chuyển dương đến trong thời hạn đều được ghi có theo đúng số tiền của nó. `suggested_amount` chỉ là gợi ý, không phải yêu cầu khớp số tiền: nó là `max(0, amount_due − pending)` được làm tròn **lên** theo số chữ số thập phân của tuyến, nên có thể lớn hơn `amount_due` tối đa một đơn vị token. Đừng cho rằng hai giá trị này bằng nhau.

**`direct_exact`** — địa chỉ Solana hoặc TRON của bạn kèm một số tiền chính xác:

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

Người mua phải gửi đúng `exact_amount` (`invoice_amount + matching_increment`) trong một lần chuyển duy nhất. Phần cộng thêm là cách khoản thanh toán được gán về hóa đơn này; tiền vẫn về tay bạn, nhưng không bao giờ được tính là tiền của hóa đơn.

Vì số tiền chính xác đó luôn phủ trọn hóa đơn, một tùy chọn `direct_exact` sẽ chuyển sang `unavailable` ngay khi hóa đơn có bất kỳ khoản thanh toán nào đã xác nhận hoặc đang chờ, kể cả khi khoản đó đến từ tuyến khác. Cũng vì vậy, hóa đơn trả trực tiếp mới thanh toán một phần thì không thể trả bù: hãy phát hành hóa đơn mới cho phần còn thiếu.

Chỉ tùy chọn có `status: "ready"` mới mang các trường thanh toán nêu trên. Tùy chọn `unavailable` chỉ mang các trường chung: nó đã ngừng phục vụ (đang bị xét duyệt thủ công, địa chỉ hay tuyến bị chặn, chuỗi tạm dừng, hết thời hạn thanh toán) và không nên hiển thị cho người mua nữa.

## Trạng thái thanh toán

Có hai trường trạng thái, và chúng trả lời hai câu hỏi khác nhau.

**`status`** là trạng thái ghi sổ chuẩn, dựa trên các khoản thanh toán đã xác nhận và việc tiền về ví: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` hoặc `review_required`. `paid`, `settling` và `settled` đều có nghĩa là người mua đã trả tiền — chúng chỉ khác nhau ở chỗ tiền đã về ví bạn đến đâu. `review_required` nghĩa là hóa đơn đang chờ xét duyệt thủ công — đây **không** phải trạng thái đã trả, nên đừng xử lý đơn hàng ngay cả khi `amount_paid` trông có vẻ đủ.

**`checkout_status`** là trạng thái hiển thị cho người thanh toán, được suy ra ở mỗi phản hồi và xét theo thứ tự sau:

| Giá trị | Ý nghĩa |
| --- | --- |
| `paid` | `status` là `paid`, `settling` hoặc `settled` |
| `confirming` | Đã có bằng chứng trên chuỗi, chưa được xác nhận |
| `expired` | Đã quá `monitoring_ends_at` |
| `open` | Có ít nhất một tùy chọn thanh toán ở trạng thái `ready` |
| `unavailable` | Mọi trường hợp còn lại: đang xét duyệt, tuyến bị chặn, hóa đơn thử nghiệm chưa trả |

**`checkout_status` không bao giờ là căn cứ để xử lý đơn hàng.** Hãy dùng webhook `invoice.paid`.

**`payment_revision`** bắt đầu từ `0` và tăng đúng một lần mỗi khi tập các khoản thanh toán đã xác nhận và ghi có thay đổi: có giao dịch mới, có khoản bị đảo ngược, có khoản thanh toán thử nghiệm mới, hoặc thời gian trên chuỗi của một giao dịch đã ghi có được đính chính. Riêng việc tiền về ví thì không làm nó tăng, và nó có thể đổi trong khi `status` không đổi. Dùng nó để loại bỏ bản chụp hóa đơn hay webhook đến sau bản mới hơn.

`amount_due` là `max(amount − amount_paid, 0)` và `amount_overpaid` là `max(amount_paid − amount, 0)`. Hãy đọc hai trường đó thay vì tự trừ tiền.

## Cửa sổ thanh toán

`monitoring_ends_at` rơi vào một ngày sau khi hóa đơn được tạo, và đó là ranh giới duy nhất. Một giao dịch chỉ được ghi có tự động nếu thời gian trên chuỗi của chính nó nằm trong cửa sổ — không tính gì trước khi hóa đơn tồn tại, cũng không tính gì đúng lúc `monitoring_ends_at` hay sau đó. Đồng hồ của bạn, thời điểm chúng tôi quan sát và thời điểm webhook đến đều không tham gia vào việc này.

Khoản thanh toán đến muộn không bị mất. Nó vẫn được ghi vào hóa đơn và hiển thị trong bảng điều khiển, nơi bạn có thể ghi có cho nó bằng cách chỉ đích danh giao dịch — một khẳng định mà không quy trình tự động nào làm thay bạn được. Bạn có bao nhiêu thời gian thì tùy tuyến:

- **EVM** — không có hạn. Địa chỉ nạp tiền chỉ thuộc về hóa đơn này và không bao giờ được cấp lại, nên một khoản chuyển đến đó không thể là tiền của hóa đơn nào khác.
- **Solana và TRON** — 21 ngày kể từ lúc tạo. Số tiền chính xác được giữ thêm 20 ngày sau `monitoring_ends_at`; sau mốc đó nó có thể đã thuộc về một hóa đơn mới hơn, và không ai khẳng định được một khoản chuyển đến muộn đã trả cho hóa đơn nào. Điều quan trọng là giao dịch đến lúc nào, chứ không phải bạn xử lý nó lúc nào.

Một hệ quả cho phần tích hợp của bạn: **`invoice.paid` có thể đến rất lâu sau khi trang thanh toán đã báo `expired`**, và không trường nào phân biệt được. Nếu bạn hủy hoặc bán lại đơn hàng ngay khi trang thanh toán hết hạn, hãy đối chiếu `invoice.paid` với trạng thái đơn hàng của chính bạn thay vì mặc định là hóa đơn còn mở — và trong mọi trường hợp hãy xử lý theo kiểu idempotent.

## Webhook

Cấu hình URL webhook trong bảng điều khiển. Thử nghiệm và chạy thật mỗi bên có URL và khóa ký riêng.

### Sự kiện

**`invoice.paid`** — hóa đơn đã đạt trạng thái đã trả (`paid`, `settling` hoặc `settled`):

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

**`invoice.payment_reversed`** — một hóa đơn từng được trả đủ nay tụt xuống dưới số tiền của nó, chẳng hạn vì chuỗi tổ chức lại và một giao dịch đã ghi có bị gỡ bỏ. Cấu trúc payload giống hệt, với `status` hiện tại của hóa đơn, `payment_revision` cao hơn và `fully_paid_at: null`. Hãy coi đó là việc vô hiệu hóa tín hiệu thanh toán trước đó, theo chính sách kinh doanh của bạn.

- `review_required` không bao giờ kích hoạt `invoice.paid`. Nếu ngưỡng bị vượt qua trong lúc xét duyệt, sự kiện chỉ được gửi một lần, sau khi xét duyệt xong.
- Một chuỗi thực sự đã trả → đảo ngược → đã trả sẽ gửi `invoice.paid`, rồi `invoice.payment_reversed`, rồi một `invoice.paid` mới, mỗi cái kèm `payment_revision` tương ứng.
- `reference_id` và `fully_paid_at` có thể là `null` nhưng luôn có mặt. `return_url` và hướng dẫn thanh toán cố ý không được gửi kèm — hãy đối soát ở phía máy chủ bằng id hóa đơn cộng `reference_id`.

### Xác minh chữ ký

Mỗi lần gửi đều kèm:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` là dấu thời gian Unix tính bằng giây. `v1` là HMAC-SHA256 dạng hex chữ thường của `<t>.<raw_body>`, dùng khóa webhook của chế độ tương ứng. Hãy xác minh trên **thân request thô** trước khi phân tích cú pháp — tuần tự hóa lại JSON có thể làm đổi byte và khiến chữ ký sai. Đồng thời hãy từ chối các dấu thời gian nằm ngoài khoảng dung sai chống phát lại của bạn. Các SDK chính thức đều có sẵn hàm `verifyWebhook`.

### Cơ chế gửi và gửi lại

- Mỗi lần gửi sẽ hết hạn sau 10 giây.
- Mọi phản hồi không phải 2xx — **kể cả chuyển hướng và 4xx** — cùng lỗi mạng và hết thời gian chờ đều được thử lại với khoảng lùi có giới hạn: 1 phút, 5 phút, 30 phút, rồi 2 giờ, mỗi lần lệch ngẫu nhiên tối đa 20%, tổng cộng năm lần.
- Việc gửi là **ít nhất một lần và có thể không đúng thứ tự.** Hãy khử trùng lặp theo `id` sự kiện, giữ lại bản chụp có `payment_revision` lớn nhất, và trả `2xx` thật nhanh — xử lý công việc sau khi đã báo nhận.

## Chế độ thử nghiệm và chuyển sang chạy thật

### `POST /v1/invoices/{id}/test-payments`

Thêm một khoản thanh toán mô phỏng vào hóa đơn **thử nghiệm** và trả về trạng thái thanh toán mới nhất. Chỉ dùng được với khóa `sk_test_...` — đây là cách bạn chạy trọn vòng chưa trả → đã trả → webhook mà không đụng đến chuỗi nào.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` là bắt buộc và phải lớn hơn không, tối đa 15 chữ số phần nguyên và 4 chữ số thập phân (`5`, `5.0` và `5.0000` đều chuẩn hóa thành `5.0000`).
- `reference_id` là tùy chọn, tối đa 200 ký tự, và idempotent theo từng hóa đơn: cùng mã tham chiếu với cùng số tiền đã chuẩn hóa sẽ trả `200 OK` và `meta.result: "reused"`; số tiền khác đi sẽ trả `409 test_payment_reference_conflict`.
- Cho phép trả một phần, trả đủ và trả dư: `partially_paid` khi `0 < amount_paid < amount`, `paid` khi `amount_paid >= amount`. Lần đầu vượt qua mốc `paid` sẽ gửi một `invoice.paid`, và mỗi khoản thanh toán mới đều làm `payment_revision` tăng.
- Giới hạn 300 lần/phút và 10.000 lần/ngày cho mỗi dự án.

Phản hồi là hóa đơn theo cấu trúc lúc tạo, cộng thêm `amount_paid` và `fully_paid_at`, kèm `meta.result`.

Mã lỗi: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

### Chuyển sang chạy thật

Khi vòng lặp đã chạy được với webhook thử nghiệm của bạn — một tunnel như ngrok hay cloudflared là đủ cho phát triển local:

1. Tạo một khóa `sk_live_` trong bảng điều khiển.
2. Đặt URL webhook chạy thật của bạn.
3. Đổi khóa trong cấu hình máy chủ.

Không gì khác thay đổi: cùng endpoint, cùng cấu trúc dữ liệu, và hóa đơn thật giờ mang `payment_options` thật. Hóa đơn thử nghiệm và thanh toán thử nghiệm không bao giờ đụng đến chuỗi và không bao giờ được tính là thanh toán thật.

## Hỗ trợ

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
