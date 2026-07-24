# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · **Tiếng Việt** · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Tài liệu này được dịch từ README tiếng Anh; nếu có chỗ khác nhau, [bản tiếng Anh](../README.md) là bản chuẩn.

Thanh toán stablecoin cho lập trình viên độc lập. Không lưu ký — tiền về thẳng ví của chính bạn.

Đây là tài liệu tham chiếu cho REST API công khai của invoq. Nếu bạn dùng một trong các SDK chính thức ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)), thì chúng chỉ là lớp bọc quanh chính những endpoint này — tài liệu này là bản hợp đồng mà các SDK tuân theo.

- **Base URL:** `https://api.invoq.money`
- **Trang thanh toán được lưu trữ sẵn:** `https://pay.invoq.money/<id hóa đơn>`
- **Bảng điều khiển** (khóa API, ví nhận tiền, webhook): `https://app.invoq.money`

## Cách hoạt động

1. **Tạo hóa đơn** từ máy chủ của bạn (`POST /v1/invoices`).
2. **Để người mua thanh toán hóa đơn.** Cách dễ nhất: gửi họ đến trang thanh toán được lưu trữ sẵn tại `https://pay.invoq.money/<id hóa đơn>` — trang hiển thị số tiền, địa chỉ và mã QR, hỗ trợ mười ngôn ngữ, và bạn không phải động tay làm chút UI nào. Hoặc nhúng chính trang thanh toán đó vào website của bạn bằng [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). Người mua gửi USDC hoặc USDT từ bất kỳ ví hay sàn giao dịch nào.
3. **Nhận thông báo khi hóa đơn được thanh toán.** invoq xác nhận khoản chuyển trên chuỗi rồi gửi webhook `invoice.paid` đến máy chủ của bạn; tiền về thẳng ví của chính bạn.

## Bắt đầu nhanh

Lấy một khóa thử nghiệm (`sk_test_...`) từ bảng điều khiển, rồi tạo hóa đơn đầu tiên của bạn:

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

Với hóa đơn thật, bạn chỉ cần `id` trong phản hồi — `https://pay.invoq.money/<id>` chính là trang thanh toán của bạn. Còn hóa đơn vừa tạo ở trên là hóa đơn thử nghiệm, không mang hướng dẫn thanh toán trên chuỗi nào, nên hãy mô phỏng thanh toán thay vì trả thật:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Nếu bạn đã lưu URL webhook thử nghiệm trong bảng điều khiển, nó sẽ nhận được một `invoice.paid` có chữ ký. Toàn bộ vòng lặp chỉ có vậy — phần còn lại của tài liệu này là chi tiết.

## Xác thực

Các endpoint phía máy chủ dùng một khóa API bí mật được tạo trong bảng điều khiển:

```http
Authorization: Bearer sk_test_...
```

- Khóa `sk_test_...` tạo **hóa đơn thử nghiệm**: thanh toán được mô phỏng, không có gì di chuyển trên chuỗi, còn webhook là thật (được ký và gửi y như khi chạy thật).
- Khóa `sk_live_...` tạo **hóa đơn thật** với hướng dẫn thanh toán trên chuỗi thật.

Chế độ của hóa đơn luôn do khóa quyết định — nội dung request không bao giờ nhận trường `mode`. Giữ khóa bí mật trên máy chủ của bạn; đừng bao giờ đưa chúng vào code phía client.

## Cấu trúc phản hồi

Phản hồi thành công bọc tài nguyên trong `data`:

```json
{ "data": { "id": "inv_..." } }
```

Phản hồi lỗi dùng chung một cấu trúc trên mọi endpoint:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` là mã lỗi ổn định dành cho máy đọc — hãy rẽ nhánh theo nó, đừng theo `message`.
- `fields` chỉ xuất hiện với lỗi kiểm tra hợp lệ ở cấp trường.
- Ngữ cảnh nghiệp vụ bổ sung được trả về trong `meta`, chẳng hạn `retry_after`, `reason_codes` hoặc `min_amount`.

Nội dung request bị giới hạn ở 4KB; nội dung quá cỡ sẽ nhận `413 request_body_too_large`.

## Tạo hóa đơn

### `POST /v1/invoices`

Tạo một hóa đơn và trả về bản tóm tắt cùng hướng dẫn thanh toán của nó.

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
| `amount` | Bắt buộc. Chuỗi thập phân, `0.01`–`1000000.00`, tối đa 2 chữ số lẻ. Được chuẩn hóa trong phản hồi (`12.34` → `12.3400`). Đơn vị tiền nghiệp vụ cố định là `USD` và được trả về trong phản hồi; nó không phải là trường của request. |
| `reference_id` | Tùy chọn, mã tham chiếu phía bên gọi, duy nhất theo từng dự án + chế độ, tối đa 200 ký tự. Thử lại với nội dung giống hệt sẽ trả về hóa đơn đã có kèm `200 OK`; nội dung khác sẽ nhận `409 reference_id_conflict`. |
| `description` | Tùy chọn, văn bản hiển thị cho người thanh toán, tối đa 500 ký tự. |
| `return_url` | Tùy chọn, URL `http(s)` hiển thị làm nút quay về trang người bán sau khi thanh toán, tối đa 1000 ký tự. Bỏ trống → URL quay về mặc định của dự án được chụp lại tại thời điểm tạo. Truyền hẳn `null` hoặc `""` → không có URL quay về. Khi thử lại theo `reference_id`, một `return_url` bị bỏ trống sẽ không được so khớp với hóa đơn đã có — hãy truyền tường minh khi lần thử lại cần khẳng định một giá trị cụ thể. |

Phản hồi thành công (`201 Created`; tạo lại trùng lặp trả về `200 OK` kèm `meta.result: "reused"`):

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

Vài điểm ngữ nghĩa đáng lưu ý:

- **Không thể đổi hướng dòng tiền qua API này.** Request không thể đặt địa chỉ nhận tiền hay cấu hình hợp đồng — chúng lấy từ cài đặt dự án đã xác minh của bạn tại thời điểm tạo hóa đơn. Sau khi tạo, hóa đơn là bất biến, chỉ trạng thái thanh toán và trạng thái tiền về ví là thay đổi.
- **`payment_options` liệt kê các cách người mua có thể trả**, mỗi mục ứng với một mạng + token, theo thứ tự hiển thị cho người thanh toán (USDT trước USDC, rồi đến thứ tự mạng đã duyệt của từng token). `collection_method` là trường phân biệt:
  - `evm_deposit` — địa chỉ nạp tiền EVM riêng cho từng hóa đơn. Mọi khoản chuyển dương đến trong thời hạn đều được ghi có theo đúng số tiền của nó; `suggested_amount` (`max(0, amount_due − pending)`) chỉ là gợi ý, không phải yêu cầu khớp số tiền.
  - `direct_exact` — địa chỉ người bán trên Solana/TRON kèm một số tiền chính xác. Người mua phải gửi đúng `exact_amount` (`invoice_amount + matching_increment`) trong một lần chuyển duy nhất; phần cộng thêm là cách khoản thanh toán được gán về đúng hóa đơn và không bao giờ được ghi có cho hóa đơn.
  - Chỉ tùy chọn có `status: "ready"` mới mang các trường thanh toán nêu trên; tùy chọn `unavailable` chỉ mang các trường chung. Hãy nhận diện một tùy chọn bằng bộ (`chain_namespace`, `chain_reference`, `token_address`), đừng bao giờ dựa vào vị trí trong mảng hay metadata hiển thị.
- **`checkout_status` là trạng thái hiển thị cho người thanh toán**, được suy ra ở mỗi phản hồi: `paid` (trạng thái chuẩn là paid/settling/settled), `confirming` (đang chờ bằng chứng trên chuỗi), `expired` (đã quá `monitoring_ends_at`), `open` (còn ít nhất một tùy chọn sẵn sàng), hoặc `unavailable`. Nó không bao giờ là căn cứ để xử lý đơn hàng — hãy dùng webhook `invoice.paid`.
- **`payment_revision`** bắt đầu từ `0` và tăng đúng một lần mỗi khi tập các khoản thanh toán đã xác nhận và ghi có thay đổi (mỗi khoản thanh toán thử nghiệm mới cũng vậy). Dùng nó để loại bỏ một bản chụp hóa đơn hay webhook cũ nhưng được gửi đến sau bản mới hơn.
- **Số tiền là chuỗi thập phân, không bao giờ là float.** Số tiền đã trả/còn thiếu dùng thang 18 chữ số thập phân. `amount_due` là `max(amount − amount_paid, 0)` và `amount_overpaid` là `max(amount_paid − amount, 0)` — hãy đọc các trường đó thay vì tự trừ tiền.
- **invoq theo dõi chuỗi trong 7 ngày** sau khi tạo (`monitoring_ends_at`). Khoản chuyển đến vào đúng hoặc sau thời điểm đó vẫn được ghi nhận nhưng không được ghi có gì; tính năng đối soát thủ công trong bảng điều khiển là chốt chặn của người vận hành cho các trường hợp hiếm bên trong thời hạn.
- Hóa đơn thử nghiệm trả về `monitoring_ends_at: null`, `payment_options: []` và `checkout_status: "unavailable"` — thanh toán được mô phỏng qua endpoint test-payments bên dưới.
- Giới hạn tần suất theo từng dự án: chạy thật 3.000/phút và 100.000/ngày; thử nghiệm 300/phút và 10.000/ngày.

Các mã lỗi: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (kèm mã lỗi cấp trường `amount_too_small` / `amount_too_large` và `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (kèm `meta.reason_codes` đã được sắp xếp: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — `merchant_address_provisioning` chỉ là tạm thời và thường tự hết sau một hai phút), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Đọc hóa đơn

### `GET /v1/invoices/{id}`

Trả về bản tóm tắt công khai của hóa đơn, trạng thái thanh toán hiển thị cho người thanh toán, nhận diện thương hiệu của dự án và hướng dẫn thanh toán. **Không cần khóa API** — id hóa đơn là id công khai chia sẻ được và không thể đoán, dùng trong các URL link thanh toán, nên đây là endpoint mà các UI thanh toán dùng để poll (CORS cho phép mọi origin với GET).

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

- `status` là trạng thái chuẩn của hóa đơn, dựa trên các sự kiện thanh toán và tiền về ví đã xác nhận: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` hoặc `review_required`.
- `review_required` nghĩa là hóa đơn đang chờ duyệt thủ công. Đây **không** phải trạng thái đã thanh toán — đừng xử lý đơn hàng dựa trên nó, kể cả khi `amount_paid` trông có vẻ đủ.
- `checkout_status` là trạng thái suy ra hiển thị cho người thanh toán như mô tả ở trên; hóa đơn thật đang chờ bằng chứng trên chuỗi sẽ hiện `confirming` trong khi `status` chuẩn giữ nguyên cho đến khi khoản chuyển được xác nhận.
- `transfers` là danh sách biên nhận dành cho người thanh toán: các khoản chuyển vào đã xác nhận và được ghi có cho hóa đơn này, để trang thanh toán có thể hiển thị từng giao dịch trên chuỗi và dẫn link tới block explorer. Mỗi mục mang `chain_namespace`, `chain_reference`, `transaction_id` chuẩn, `event_index`, `amount` (theo đơn vị tiền của hóa đơn, cùng thang 18 chữ số thập phân như `amount_paid`; với `direct_exact` thì không gồm phần cộng thêm để khớp) và `explorer_transaction_url` (hoặc `null`). Chỉ khoản chuyển đã xác nhận mới xuất hiện — khoản đang chờ vẫn có thể bị một reorg loại bỏ — và bị giới hạn ở 20 khoản lớn nhất theo số tiền, để những khoản dust gửi vào một địa chỉ nạp tiền công khai không thể chèn mất khoản thanh toán thật. Luôn có mặt: là `[]` cho đến khi có khoản chuyển được xác nhận, và luôn là `[]` với hóa đơn thử nghiệm.
- `reference_id` vốn chỉ dành cho bên gọi được lược bỏ ở đây; chỉ các trường nhận diện thương hiệu `project` dành cho người thanh toán được trả về.
- Id sai định dạng lẫn id không tồn tại đều trả về `404 invoice_not_found`.

## Mô phỏng thanh toán thử nghiệm

### `POST /v1/invoices/{id}/test-payments`

Thêm một khoản thanh toán mô phỏng vào hóa đơn **thử nghiệm** và trả về trạng thái thanh toán đã cập nhật. Chỉ dùng được với khóa `sk_test_...` — đây là cách bạn chạy trọn vòng lặp unpaid → paid → webhook mà không đụng đến chuỗi nào.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` là bắt buộc, phải lớn hơn 0, tối đa 15 chữ số phần nguyên và 4 chữ số lẻ (`5`, `5.0`, `5.0000` đều chuẩn hóa thành `5.0000`).
- `reference_id` là tùy chọn, tối đa 200 ký tự, và an toàn khi thử lại theo từng hóa đơn: dùng lại nó với cùng số tiền đã chuẩn hóa sẽ trả về `200 OK` kèm `meta.result: "reused"`; số tiền khác sẽ nhận `409 test_payment_reference_conflict`.
- Cho phép trả một phần, trả đủ và trả dư: `partially_paid` khi `0 < amount_paid < amount`, `paid` khi `amount_paid >= amount`. Lần đầu chuyển sang `paid` kích hoạt đúng một webhook `invoice.paid` về mặt logic, và mỗi khoản thanh toán được tạo sẽ tăng `payment_revision`.
- Việc tạo bị giới hạn 300 lần mỗi phút và 10.000 lần mỗi ngày cho từng dự án.

Các mã lỗi: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhook

Cấu hình URL webhook trong bảng điều khiển — chế độ thử nghiệm và chạy thật mỗi bên có URL và mã bí mật ký riêng.

### Sự kiện

**`invoice.paid`** — gửi khi hóa đơn chuyển sang trạng thái đã thanh toán (`paid`, `settling` hoặc `settled`):

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

**`invoice.payment_reversed`** — gửi khi một hóa đơn từng thanh toán đủ tụt trở lại xuống dưới giá trị của nó (ví dụ một reorg trên chuỗi đã gỡ bỏ khoản chuyển từng được ghi có). Payload cùng cấu trúc, với `status` hiện tại của hóa đơn, `amount_paid`, một `payment_revision` cao hơn và `fully_paid_at: null`. Hãy coi nó là tín hiệu vô hiệu hóa tín hiệu xử lý đơn hàng trước đó, và xử trí theo chính sách kinh doanh của riêng bạn.

- `review_required` không bao giờ kích hoạt `invoice.paid`. Chỉ khi duyệt xong và hóa đơn chuyển sang trạng thái đã thanh toán thì sự kiện mới được tạo.
- Một chuỗi thật sự kiểu đã trả → bị đảo ngược → trả lại sẽ gửi `invoice.paid`, `invoice.payment_reversed`, rồi một `invoice.paid` mới, mỗi sự kiện kèm `payment_revision` tương ứng.
- `reference_id` và `fully_paid_at` có thể là null nhưng luôn có mặt; `return_url` và hướng dẫn thanh toán được cố ý lược bỏ. Hãy đối soát phía máy chủ bằng id hóa đơn cùng `reference_id`.

### Xác minh chữ ký

Mọi webhook gửi đi đều kèm:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` là timestamp Unix tính bằng giây. `v1` là HMAC-SHA256 dạng hex chữ thường của `<t>.<raw_body>`, ký bằng mã bí mật webhook riêng của từng chế độ. Hãy xác minh trên **nội dung request gốc** trước khi phân tích — việc serialize lại JSON có thể làm đổi các byte và khiến chữ ký mất hiệu lực. Từ chối những timestamp nằm ngoài khoảng dung sai chống phát lại của bạn. (Các SDK chính thức có sẵn hàm hỗ trợ `verifyWebhook`.)

### Cơ chế gửi và gửi lại

- Request POST gửi webhook sẽ timeout sau 10 giây.
- Mọi phản hồi không phải 2xx — **kể cả redirect và 4xx** — cùng với lỗi mạng và timeout đều được gửi lại với khoảng chờ tăng dần có giới hạn: 1 phút, 5 phút, 30 phút, rồi 2 giờ, mỗi lần cộng thêm tối đa 20% jitter, tổng cộng tối đa năm lần thử.
- Việc gửi là **ít nhất một lần** và có thể sai thứ tự: hãy khử trùng lặp theo `id` sự kiện, giữ bản chụp có `data.invoice.payment_revision` lớn nhất, và trả về `2xx` thật nhanh (xử lý công việc sau khi đã báo nhận).

## Chuyển sang chạy thật

Khi vòng lặp trong phần [Bắt đầu nhanh](#bắt-đầu-nhanh) đã chạy được với webhook thử nghiệm của bạn (một tunnel như ngrok hay cloudflared là đủ cho phát triển local):

1. Tạo một khóa `sk_live_` trong bảng điều khiển.
2. Lưu URL webhook chạy thật của bạn trong bảng điều khiển.
3. Đổi khóa trong cấu hình máy chủ của bạn. Không gì khác thay đổi: cùng endpoint, cùng cấu trúc dữ liệu — hóa đơn thật giờ mang `payment_options` thật.

Hóa đơn thử nghiệm và thanh toán thử nghiệm không bao giờ đụng đến chuỗi và không bao giờ được tính là thanh toán thật.

## Hỗ trợ

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
