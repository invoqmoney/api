# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · **简体中文** · [繁體中文](./README.zh-Hant.md)

> 本文是英文版 README 的简体中文翻译；若表述有出入，以[英文版](../README.md)为准。

面向独立开发者的稳定币收款。非托管——资金直接结算进你自己的钱包。

这里是 invoq 公开 REST API 的参考文档。如果你在用官方 SDK（[Node.js](https://github.com/invoqmoney/sdk-js)、[Python](https://github.com/invoqmoney/sdk-python)、[PHP](https://github.com/invoqmoney/sdk-php)、[Go](https://github.com/invoqmoney/sdk-go)、[Rust](https://github.com/invoqmoney/sdk-rust)、[Ruby](https://github.com/invoqmoney/sdk-ruby)），它们封装的正是这里的接口——本文档就是它们遵循的契约。

- **Base URL：**`https://api.invoq.money`
- **托管收银页：**`https://pay.invoq.money/<账单 id>`
- **商户后台**（API 密钥、收款钱包、webhook）：`https://app.invoq.money`

## 工作流程

1. **在服务端创建账单**（`POST /v1/invoices`）。
2. **让买家付款。** 最省事的做法是把买家引导到托管收银页 `https://pay.invoq.money/<账单 id>`——页面会展示金额、地址和二维码，支持十种语言，你一行 UI 代码都不用写。也可以用 [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) 把同一个收银台嵌进自己的网站。买家用任意钱包或交易所转 USDC 或 USDT 即可。
3. **付款完成时收到通知。** invoq 在链上确认转账后，向你的服务器发送 `invoice.paid` webhook；结算款直接进你自己的钱包。

## 快速开始

先在商户后台拿一把测试密钥（`sk_test_...`），然后创建你的第一张账单：

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

对正式账单来说，响应里的 `id` 就是你需要的全部——`https://pay.invoq.money/<id>` 就是你的收银页。上面这张是测试账单，不带链上付款信息，所以改用模拟付款：

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

如果你在商户后台设置了测试 webhook URL，它会收到一条带签名的 `invoice.paid`。整个闭环到此就跑通了——本文档余下的部分都是细节。

## 身份认证

服务端接口使用在商户后台创建的 API 密钥（secret key）：

```http
Authorization: Bearer sk_test_...
```

- `sk_test_...` 密钥创建的是**测试账单**：付款是模拟的，链上不会发生任何转账，但 webhook 是真实的（和正式环境一样签名、一样投递）。
- `sk_live_...` 密钥创建的是**正式账单**，带真实的链上付款信息。

账单的模式始终由密钥决定——请求体从不接受 `mode` 字段。密钥只放在服务端；绝不要打包进客户端代码。

## 响应结构

成功响应把资源包在 `data` 里：

```json
{ "data": { "id": "inv_..." } }
```

所有接口的错误响应共用同一种结构：

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` 是稳定的、机器可读的错误码——代码分支请依据它，而不是 `message`。
- `fields` 只在字段级校验错误时出现。
- 额外的业务上下文放在 `meta` 里返回，例如 `retry_after`、`reason_codes` 或 `min_amount`。

请求体上限为 4KB；超限会返回 `413 request_body_too_large`。

## 创建账单

### `POST /v1/invoices`

创建一张账单，返回其摘要和付款信息。

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| 字段 | 说明 |
| --- | --- |
| `amount` | 必填。十进制字符串，`0.01`–`1000000.00`，最多 2 位小数。响应中会做归一化（`12.34` → `12.3400`）。业务币种固定为 `USD` 并在响应中返回；它不是请求字段。 |
| `reference_id` | 可选的调用方侧关联标识，在同一项目 + 模式内唯一，最长 200 字符。以完全相同的条款重试会返回已有账单和 `200 OK`；条款不同则返回 `409 reference_id_conflict`。 |
| `description` | 可选，买家可见的文字说明，最长 500 字符。 |
| `return_url` | 可选的 `http(s)` URL，付款完成后作为返回商家的按钮展示，最长 1000 字符。省略时，账单会快照项目当前的默认 return URL；显式传 `null` 或 `""` 则不设返回链接。用 `reference_id` 重试时，省略的 `return_url` 不会与已有账单比对——如果重试必须断言某个具体值，请显式传入。 |

成功响应（`201 Created`；幂等复用返回 `200 OK`，并带 `meta.result: "reused"`）：

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

几条值得了解的语义：

- **资金去向无法通过这个 API 改变。** 请求不能设置收款地址或合约配置——它们在账单创建时取自你已验证的项目设置。账单一经创建即不可变，只有付款和结算状态会变化。
- **`payment_options` 列出买家可用的付款方式**，每个网络 + 代币一条，按买家侧顺序排列（USDT 在 USDC 之前，其后是各代币经审核的网络顺序）。`collection_method` 是判别字段：
  - `evm_deposit`——每张账单专属的 EVM 收款地址。任何在时限内到账的正数转账都按其金额入账；`suggested_amount`（`max(0, amount_due − pending)`）只是建议值，不是匹配要求。
  - `direct_exact`——Solana/TRON 商户地址搭配一个精确金额。买家必须在单笔转账中恰好转入 `exact_amount`（`invoice_amount + matching_increment`）；该增量只用于把付款归属到账单，绝不会计入账单金额。
  - 只有 `status: "ready"` 的选项才携带上述可付款字段；`unavailable` 的选项只携带公共字段。识别一个选项要用（`chain_namespace`、`chain_reference`、`token_address`）组合，绝不要靠数组位置或展示元数据。
- **`checkout_status` 是面向买家的状态**，每次响应时都会重新派生：`paid`（权威状态为 paid/settling/settled）、`confirming`（存在待确认的链上凭证）、`expired`（已过 `monitoring_ends_at`）、`open`（至少有一个就绪的付款选项）或 `unavailable`。它永远不能作为履约依据——履约请以 `invoice.paid` webhook 为准。
- **`payment_revision`** 从 `0` 开始，每当已确认入账的付款集合发生变化时恰好递增一次（每笔新的测试付款同样如此）。用它来丢弃比更新版本晚到的旧账单快照或旧 webhook。
- **金额一律是十进制字符串，绝不是浮点数。** 已付/待付金额使用 18 位小数。`amount_due` 即 `max(amount − amount_paid, 0)`，`amount_overpaid` 即 `max(amount_paid − amount, 0)`——直接读这两个字段，不要自己做金额减法。
- **invoq 在账单创建后监控链上 7 天**（`monitoring_ends_at`）。恰在该时刻或之后落账的转账只记录、不入账；窗口内的边缘情况，以商户后台的手动对账作为运营兜底。
- 测试账单返回 `monitoring_ends_at: null`、`payment_options: []` 和 `checkout_status: "unavailable"`——付款通过下文的 test-payments 接口模拟。
- 速率限制按项目计：正式 3,000 次/分钟、100,000 次/天；测试 300 次/分钟、10,000 次/天。

错误码：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`（附 `amount_too_small` / `amount_too_large` 字段码和 `meta.min_amount` / `meta.max_amount`）、`409 reference_id_conflict`、`409 project_archived`、`409 no_payment_options_available`（附排序后的 `meta.reason_codes`：`no_merchant_address`、`merchant_address_provisioning`、`below_rail_minimum`、`rail_unavailable`、`scanner_unavailable`、`scanner_capacity_exhausted`、`matching_capacity_exhausted`——其中 `merchant_address_provisioning` 是暂时性的，通常一两分钟内就会消失）、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

## 查询账单

### `GET /v1/invoices/{id}`

返回公开的账单摘要、买家可见的付款状态、项目品牌信息和付款信息。**无需 API 密钥**——账单 id 是可分享、无法猜测的公开 id，会出现在付款链接 URL 里，所以支付界面轮询的正是这个接口（GET 的 CORS 允许任意来源）。

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

- `status` 是账单的权威状态，由已确认的付款和结算事件支撑：`unpaid`、`partially_paid`、`paid`、`settling`、`settled` 或 `review_required`。
- `review_required` 表示账单正在等待人工审核。它**不是**已付款状态——即使 `amount_paid` 看起来够了，也不要履约。
- `checkout_status` 是上文描述的派生买家侧状态；存在待确认链上凭证的正式账单会显示 `confirming`，权威 `status` 则保持不变，直到转账确认为止。
- `transfers` 是面向买家的收款回执记录：已确认并计入该账单的入账转账，收银台可以据此展示每笔链上交易并链接到区块浏览器。每条记录带 `chain_namespace`、`chain_reference`、规范化的 `transaction_id`、`event_index`、`amount`（账单币种单位，与 `amount_paid` 相同的 18 位小数 scale；`direct_exact` 时不含匹配增量）和 `explorer_transaction_url`（或 `null`）。只有已确认的转账才会出现——待确认的转账仍可能被链重组丢弃——并按金额取最大的 20 笔封顶，以免打到公开收款地址上的粉尘把真正的付款挤出列表。该字段始终存在：在有转账确认之前是 `[]`，测试账单则永远是 `[]`。
- 仅供调用方使用的 `reference_id` 在这里不返回；只返回面向买家的 `project` 品牌字段。
- 格式不合法的 id 和不存在的 id 都返回 `404 invoice_not_found`。

## 模拟测试付款

### `POST /v1/invoices/{id}/test-payments`

给**测试**账单添加一笔模拟付款，返回更新后的付款状态。只能用 `sk_test_...` 密钥调用——不碰链就能完整走一遍 unpaid → paid → webhook 闭环，靠的就是这个接口。

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` 必填，必须大于零，整数部分最多 15 位、小数部分最多 4 位（`5`、`5.0`、`5.0000` 都归一化为 `5.0000`）。
- `reference_id` 可选，最长 200 字符，按账单幂等：用相同的归一化金额复用它，返回 `200 OK` 和 `meta.result: "reused"`；金额不同则返回 `409 test_payment_reference_conflict`。
- 部分付款、足额付款和超额付款都允许：`0 < amount_paid < amount` 时为 `partially_paid`，`amount_paid >= amount` 后为 `paid`。首次进入 `paid` 触发一条逻辑上的 `invoice.paid` webhook，每笔创建的付款都会让 `payment_revision` 加一。
- 创建操作按项目限制为 300 次/分钟、10,000 次/天。

错误码：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`、`404 invoice_not_found`、`409 project_archived`、`409 test_mode_required`、`409 test_payment_reference_conflict`、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

## Webhook

在商户后台配置 webhook URL——测试和正式各有独立的 URL 和签名密钥。

### 事件

**`invoice.paid`**——账单进入已付款状态（`paid`、`settling` 或 `settled`）时发送：

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

**`invoice.payment_reversed`**——先前已付款的账单重新跌回其金额以下时发送（例如链重组移除了一笔已入账的转账）。载荷结构相同，携带账单当前的 `status`、`amount_paid`、更高的 `payment_revision`，以及 `fully_paid_at: null`。请把它视为对先前履约信号的作废，具体处置按你自己的业务策略来。

- `review_required` 绝不会触发 `invoice.paid`。只有审核通过、进入已付款状态之后，事件才会创建。
- 如果真的发生了付款 → 冲正 → 再次付款，会依次投递 `invoice.paid`、`invoice.payment_reversed`，然后是一条新的 `invoice.paid`，各自带上对应结果的 `payment_revision`。
- `reference_id` 和 `fully_paid_at` 可为 null，但一定存在；`return_url` 和付款信息则有意不包含。服务端对账请用账单 id 加 `reference_id`。

### 验证签名

每条发出的 webhook 都包含：

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` 是 Unix 秒级时间戳。`v1` 是用对应模式的 webhook 签名密钥对 `<t>.<raw_body>` 计算出的小写十六进制 HMAC-SHA256。请在解析之前用**原始请求体**验签——重新序列化 JSON 可能改变字节，让签名失效。超出你重放容忍窗口的时间戳，直接拒绝。（官方 SDK 内置了 `verifyWebhook` 辅助函数。）

### 投递与重试

- 投递 POST 的超时时间为 10 秒。
- 所有非 2xx 响应——**包括重定向和 4xx**——加上网络错误和超时，都会按有界退避重试：1 分钟、5 分钟、30 分钟、再到 2 小时，每档最多加 20% 抖动，总计最多五次尝试。
- 投递语义是**至少一次**，且可能乱序：按事件 `id` 去重，保留 `data.invoice.payment_revision` 最大的快照，并尽快返回 `2xx`（先确认，再处理业务）。

## 正式上线

当[快速开始](#快速开始)里的闭环已经能打到你的测试 webhook 时（本地开发用 ngrok 或 cloudflared 这类隧道即可）：

1. 在商户后台创建一把 `sk_live_` 密钥。
2. 在商户后台设置正式环境的 webhook URL。
3. 在服务端配置里换上这把密钥。其余一切不变：同样的接口、同样的结构——现在正式账单会带上真实的 `payment_options`。

测试账单和测试付款永远不碰链，也永远不会被算作真实付款。

## 支持

- X：中文 [@invoqcn](https://x.com/invoqcn) · English [@invoqmoney](https://x.com/invoqmoney)
- Discord：https://discord.gg/V8cVrg4dET
- Telegram：https://t.me/invoqmoney
- 邮箱：help@invoq.money
