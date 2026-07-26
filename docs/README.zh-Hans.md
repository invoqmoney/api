# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · **简体中文** · [繁體中文](./README.zh-Hant.md)

> 本文是英文版 README 的简体中文翻译；若表述有出入，以[英文版](../README.md)为准。

面向独立开发者的稳定币收款。非托管——资金直接结算进你自己的钱包。

这里是 invoq 公开 REST API 的参考文档。官方 SDK（[Node.js](https://github.com/invoqmoney/sdk-js)、[Python](https://github.com/invoqmoney/sdk-python)、[PHP](https://github.com/invoqmoney/sdk-php)、[Go](https://github.com/invoqmoney/sdk-go)、[Rust](https://github.com/invoqmoney/sdk-rust)、[Ruby](https://github.com/invoqmoney/sdk-ruby)）封装的正是这里的接口。

- **Base URL：**`https://api.invoq.money`
- **托管收银页：**`https://pay.invoq.money/<账单 id>`
- **商户后台**（API 密钥、收款钱包、webhook）：`https://app.invoq.money`

## 工作流程

1. **在服务端创建账单**（`POST /v1/invoices`）。
2. **让买家付款。** 最省事的做法是把买家引导到 `https://pay.invoq.money/<账单 id>`——页面会展示金额、地址和二维码，支持十种语言，你一行 UI 代码都不用写。也可以用 [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) 把同一个收银台嵌进自己的网站。买家用任意钱包或交易所转 USDT 或 USDC 即可。
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

- `sk_test_...` 密钥创建的是**测试账单**：付款是模拟的，链上不会发生任何转账，但 webhook 是真实的——和正式环境一样签名、一样投递。
- `sk_live_...` 密钥创建的是**正式账单**，带真实的链上付款信息。

模式始终由密钥决定，请求体从不接受 `mode` 字段。密钥只放在服务端，绝不要打包进客户端代码。

`GET /v1/invoices/{id}` 不需要密钥。账单 id 是无法猜测的公开 id，会出现在付款链接里，所以收银界面可以直接在浏览器里轮询它——`GET` 和 `HEAD` 的 CORS 允许任意来源。

## 请求与响应

成功的响应把资源包在 `data` 里：

```json
{ "data": { "id": "inv_..." } }
```

所有接口的错误都是同一种结构：

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- 请判断 `code`。`message` 是给人看的，随时可能改动。
- `fields` 只在字段级校验错误时出现；`location` 为 `body`、`query` 或 `path`。
- 额外的业务信息放在 `meta` 里，比如 `retry_after`、`reason_codes`、`min_amount`。

有几条约定值得先知道：

- **校验是严格的。** 无法识别的请求体字段或查询参数会返回 `400 invalid_request`，并带 `fields[].code: "unknown_field"`——给请求加一个防缓存参数就会让整个请求失败。
- **金额一律是十进制字符串，绝不是浮点数。** 账单金额 4 位小数，已付/待付金额 18 位小数，`payment_options` 里的代币金额则严格等于 `token_decimals` 位。
- **`429 rate_limited` 把重试提示放在响应体里**，即 `meta.retry_after`，单位为整秒。不会发送 `Retry-After` 响应头——请读 `meta`。
- 请求体上限 4KB，超过返回 `413 request_body_too_large`。
- 每个 JSON 响应都带 `Cache-Control: no-store`——付款状态靠轮询，任何环节都不能给你一份过期的账单。
- `GET /` 是无需认证的存活探针，返回 `204 No Content`。

创建账单的速率限制按项目计：正式 3,000 次/分钟、100,000 次/天，测试 300 次/分钟、10,000 次/天。

## 创建账单

### `POST /v1/invoices`

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
| `amount` | 必填。十进制字符串，范围 `0.01`–`1000000.00`，最多 2 位小数。响应中会归一化（`12.34` → `12.3400`）。币种固定为 `USD` 并在响应中返回，不是请求字段。 |
| `reference_id` | 可选，你自己的业务单号，在同一项目 + 同一模式下唯一，最长 200 字符。用相同条件重试会返回已存在的账单，状态码 `200 OK`、`meta.result: "reused"`；条件不同则返回 `409 reference_id_conflict`。 |
| `description` | 可选，买家可见文本，最长 500 字符。 |
| `return_url` | 可选 `http(s)` URL，最长 1000 字符——付款完成后展示的返回商户按钮。省略则快照项目的默认值；传 `null` 或 `""` 表示不要返回链接。用 `reference_id` 重试时，省略的 `return_url` 不会和已有账单做校验，所以当重试需要断言某个值时请显式传入。 |

`201 Created`，幂等复用时为 `200 OK`：

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

- **资金去向无法通过这个 API 改变。** 请求不能设置收款地址或合约配置——它们取自你已验证的项目设置。账单一经创建即不可变，只有付款和结算状态会变化。
- 测试账单返回 `monitoring_ends_at: null`、`payment_options: []` 和 `checkout_status: "unavailable"`，请用 [test-payments 接口](#post-v1invoicesidtest-payments)付款。

错误码：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`（字段码为 `invalid_format`，或 `amount_too_small` / `amount_too_large` 并附 `meta.min_amount` / `meta.max_amount`）、`409 reference_id_conflict`、`409 project_archived`、`409 no_payment_options_available`、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

`409 no_payment_options_available` 表示没能为这张账单生成任何付款方式，并附排序后的 `meta.reason_codes`：`no_merchant_address`、`merchant_address_provisioning`、`below_rail_minimum`、`rail_unavailable`、`scanner_unavailable`、`scanner_capacity_exhausted`、`matching_capacity_exhausted`。其中 `merchant_address_provisioning` 是暂时性的——新增的 Solana 或 TRON 地址仍在激活中，同样的请求通常几秒后就会成功。

## 查询账单

### `GET /v1/invoices/{id}`

账单的买家视角：摘要、付款状态、项目品牌信息、付款信息和收款回执记录。无需 API 密钥——收银界面轮询的正是这个接口。

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

相比创建响应，这里多了 `amount_paid`、`project` 和 `transfers`，并且不返回仅供调用方使用的 `reference_id`。`description`、`return_url`、`project.name`、`project.logo_url` 都可能为 `null`——收银界面在它们全都缺失时也要能正常渲染。

`transfers` 是收款回执记录，收银台可以据此展示每笔链上交易并链接到区块浏览器。每条记录带 `chain_namespace`、`chain_reference`、`transaction_id`、`event_index`、`amount`（账单币种，与 `amount_paid` 相同的 18 位小数；`direct_exact` 时不含匹配增量），以及 `explorer_transaction_url`（没有配置浏览器时为 `null`）。

只有已确认的转账才会出现（待确认的仍可能被链重组丢弃），并按金额取最大的 20 笔封顶、从大到小排列，以免粉尘把真正的付款挤出列表。该字段始终存在：在有转账确认之前是 `[]`，测试账单则永远是 `[]`。

格式不合法的 id 和不存在的 id 都返回 `404 invoice_not_found`。

## 买家怎么付款

`payment_options` 列出这张账单可用的付款方式，每个网络 + 代币一条，按买家侧顺序排列：USDT 在 USDC 之前，其后是各代币的网络顺序。有哪些选项、以及其中的地址和金额，在账单创建时就固定下来——你后来配置的收款地址或通道不会改写已有账单。只有每个选项的 `status` 会在每次响应时重新计算。

识别一个选项要用（`chain_namespace`、`chain_reference`、`token_address`）组合，绝不要靠数组位置，也不要靠 `network_label`、`display_symbol`、`logo_url`、`chain_logo_url`——这些只是展示用的元数据。

`collection_method` 是判别字段。

**`evm_deposit`**——这张账单专属的收款地址：

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

任何在时限内到账的正数转账都按其金额入账。`suggested_amount` 只是建议值，不是匹配要求：它是 `max(0, amount_due − pending)` 按该通道的小数位**向上**取整，因此可能比 `amount_due` 多出最多一个代币最小单位。不要断言两者相等。

**`direct_exact`**——你的 Solana 或 TRON 地址搭配一个精确金额：

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

买家必须在单笔转账中恰好转入 `exact_amount`（`invoice_amount + matching_increment`）。该增量用于把付款归属到这张账单；钱会到你手上，但绝不会计入账单金额。

正因为这个精确金额始终覆盖整张账单，只要账单出现任何已确认或待确认的付款——哪怕来自另一条通道——`direct_exact` 选项就会变成 `unavailable`。同理，部分付款的直转账单无法补差：请为余额另开一张账单。

只有 `status: "ready"` 的选项才携带上述可付款字段。`unavailable` 的选项只带公共字段——它已经停用（人工审核、地址或通道被封禁、链暂停处理、付款时间窗口已过），不应再展示给买家。

## 付款状态

两个状态字段，回答的是不同的问题。

**`status`** 是权威的记账状态，由已确认的付款和结算支撑：`unpaid`、`partially_paid`、`paid`、`settling`、`settled` 或 `review_required`。其中 `paid`、`settling`、`settled` 都表示买家已付款，区别只在于资金到你钱包的进度。`review_required` 表示账单正在等待人工审核——它**不是**已付款状态，即使 `amount_paid` 看起来够了也不要履约。

**`checkout_status`** 是面向买家的状态，每次响应都会重新派生，按以下顺序判定：

| 取值 | 含义 |
| --- | --- |
| `paid` | `status` 为 `paid`、`settling` 或 `settled` |
| `confirming` | 链上凭证已到达，尚未确认 |
| `expired` | 已过 `monitoring_ends_at` |
| `open` | 至少有一个 `ready` 的付款选项 |
| `unavailable` | 其余情况——审核中、通道被封、未付款的测试账单 |

**`checkout_status` 永远不能作为履约依据。** 履约请以 `invoice.paid` webhook 为准。

**`payment_revision`** 从 `0` 开始，每当已确认入账的付款集合发生变化时恰好递增一次：新增转账、发生冲正、新增测试付款，或某笔已入账转账的链上时间被修正。仅结算不会让它变化，而且它可能在 `status` 不变时改变。用它来丢弃比更新版本晚到的账单快照或 webhook。

`amount_due` 即 `max(amount − amount_paid, 0)`，`amount_overpaid` 即 `max(amount_paid − amount, 0)`。直接读这两个字段，不要自己做金额减法。

## 付款时间窗口

`monitoring_ends_at` 是账单创建后的 1 天，也是唯一的边界。只有链上时间落在窗口内的转账才会被自动入账——账单存在之前的不算，恰在 `monitoring_ends_at` 或之后的也不算。你的时钟、我们的观测时间、webhook 到达时间都不参与判断。

迟到的付款并不会丢。它会被记录在账单上并显示在商户后台，你可以在那里指明这笔交易来为它入账——这个断言是任何自动流程都无法替你做出的。你有多长时间取决于通道：

- **EVM**——没有期限。收款地址只属于这一张账单，永不重发，因此转到该地址的款项不可能属于别的账单。
- **Solana 和 TRON**——自创建起 21 天。精确金额会在 `monitoring_ends_at` 之后继续保留 20 天；过了这个期限，它可能已经属于更新的账单，谁也说不清一笔迟到的转账付的是哪一张。判断依据是转账落账的时间，而不是你何时去处理。

这对你的集成有一个直接影响：**`invoice.paid` 可能在收银页已显示 `expired` 很久之后才到达**，而且没有任何字段能把它区分出来。如果你在收银页过期时就取消订单或把货再卖一次，请拿 `invoice.paid` 和你自己的订单状态对账，而不要假设账单还开着——无论如何都要做幂等处理。

## Webhook

在商户后台配置 webhook URL。测试和正式各有自己的 URL 和签名密钥。

### 事件

**`invoice.paid`**——账单进入已付款状态（`paid`、`settling` 或 `settled`）：

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

**`invoice.payment_reversed`**——原本已付款的账单又跌回金额之下，例如链重组移除了一笔已入账的转账。载荷结构相同，带账单当前的 `status`、更高的 `payment_revision` 和 `fully_paid_at: null`。请按你自己的业务政策，把它当作此前履约信号的失效。

- `review_required` 永远不会触发 `invoice.paid`。若阈值是在审核期间被跨过的，事件会在审核通过后只发一次。
- 真实的 已付 → 冲正 → 已付 序列会依次投递 `invoice.paid`、`invoice.payment_reversed`、再一条 `invoice.paid`，各自带上对应的 `payment_revision`。
- `reference_id` 和 `fully_paid_at` 可能为 `null`，但一定存在。`return_url` 和付款信息刻意不包含在内——请在服务端用账单 id 加 `reference_id` 对账。

### 验证签名

每次投递都带：

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` 是 Unix 时间戳（秒）。`v1` 是用对应模式的 webhook 密钥对 `<t>.<raw_body>` 做 HMAC-SHA256 后的小写十六进制值。请在解析之前对**原始请求体**验签——把 JSON 重新序列化会改变字节，从而让签名失效。同时拒绝超出你重放容忍窗口的时间戳。官方 SDK 都提供了 `verifyWebhook` 辅助函数。

### 投递与重试

- 投递请求 10 秒超时。
- 任何非 2xx 响应——**包括重定向和 4xx**——以及网络错误和超时都会重试，退避间隔为 1 分钟、5 分钟、30 分钟、2 小时，每次带最多 20% 抖动，总共 5 次尝试。
- 投递是**至少一次的，且可能乱序。** 请按事件 `id` 去重，保留 `payment_revision` 最大的那份快照，并尽快返回 `2xx`——先应答，后处理。

## 测试模式与正式上线

### `POST /v1/invoices/{id}/test-payments`

给**测试**账单添加一笔模拟付款，并返回更新后的付款状态。仅 `sk_test_...` 密钥可用——这是你不碰链就跑通 未付款 → 已付款 → webhook 全流程的方式。

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` 必填且大于零，最多 15 位整数和 4 位小数（`5`、`5.0`、`5.0000` 都归一化为 `5.0000`）。
- `reference_id` 可选，最长 200 字符，在同一账单内幂等：相同单号 + 相同归一化金额返回 `200 OK` 和 `meta.result: "reused"`，金额不同则返回 `409 test_payment_reference_conflict`。
- 部分付款、全额付款和超额付款都允许：`0 < amount_paid < amount` 时为 `partially_paid`，`amount_paid >= amount` 时为 `paid`。首次跨入 `paid` 会发出一条 `invoice.paid`，每笔新付款都会让 `payment_revision` 递增。
- 限制为每项目 300 次/分钟、10,000 次/天。

响应是创建接口的账单结构，另加 `amount_paid` 和 `fully_paid_at`，并带 `meta.result`。

错误码：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`、`404 invoice_not_found`、`409 project_archived`、`409 test_mode_required`、`409 test_payment_reference_conflict`、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

### 正式上线

当整个闭环在你的测试 webhook 上跑通之后（本地开发用 ngrok 或 cloudflared 这类隧道就够了）：

1. 在商户后台创建一把 `sk_live_` 密钥。
2. 设置正式环境的 webhook URL。
3. 在服务端配置里换上这把密钥。

其他什么都不用改：接口一样、结构一样，只是正式账单现在带上了真实的 `payment_options`。测试账单和测试付款永远不触碰链，也永远不会被计为真实收款。

## 支持

- X：[@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord：https://discord.gg/V8cVrg4dET
- Telegram：https://t.me/invoqmoney
- 邮箱：help@invoq.money
