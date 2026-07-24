# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · **繁體中文**

> 本文是英文版 README 的繁體中文翻譯；若表述有出入，以[英文版](../README.md)為準。

為獨立開發者打造的穩定幣收款。非託管——資金直接結算到你自己的錢包。

這是 invoq 公開 REST API 的參考文件。如果你使用官方 SDK（[Node.js](https://github.com/invoqmoney/sdk-js)、[Python](https://github.com/invoqmoney/sdk-python)、[PHP](https://github.com/invoqmoney/sdk-php)、[Go](https://github.com/invoqmoney/sdk-go)、[Rust](https://github.com/invoqmoney/sdk-rust)、[Ruby](https://github.com/invoqmoney/sdk-ruby)），它們包裝的正是這些端點——本文件就是它們遵循的契約。

- **Base URL**：`https://api.invoq.money`
- **託管結帳頁**：`https://pay.invoq.money/<invoice id>`
- **商家後台**（API 金鑰、收款錢包、webhook）：`https://app.invoq.money`

## 運作方式

1. 在你的伺服器上**建立帳單**（`POST /v1/invoices`）。
2. **讓買家付款。**最簡單的做法：把買家導向託管結帳頁 `https://pay.invoq.money/<invoice id>`——它會顯示金額、位址和 QR code，支援十種語言，你完全不用寫 UI。也可以用 [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) 把同一個結帳頁嵌進自己的網站。買家從任何錢包或交易所轉出 USDC 或 USDT 即可。
3. **付款完成時收到通知。**invoq 在鏈上確認轉帳後，會向你的伺服器送出 `invoice.paid` webhook；款項直接結算到你自己的錢包。

## 快速開始

先到商家後台取得一組測試金鑰（`sk_test_...`），然後建立你的第一張帳單：

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

若是正式帳單，拿到回應裡的 `id` 就夠了——`https://pay.invoq.money/<id>` 就是你的結帳頁。而這張是測試帳單，不帶鏈上付款指示，所以改為模擬付款：

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

如果你已在商家後台設定測試 webhook URL，它會收到一條帶簽章的 `invoice.paid`。完整的流程就是這樣——本文件其餘的部分都是細節。

## 身分驗證

伺服器端點使用在商家後台建立的私密 API 金鑰（secret key）：

```http
Authorization: Bearer sk_test_...
```

- `sk_test_...` 金鑰建立**測試帳單**：付款是模擬的，鏈上沒有任何資金移動，但 webhook 是真實的（簽章與投遞方式和正式環境完全相同）。
- `sk_live_...` 金鑰建立**正式帳單**，帶有真實的鏈上付款指示。

帳單模式一律由金鑰決定——請求內容不接受 `mode` 欄位。私密金鑰只放在你的伺服器上，絕不要放進用戶端程式碼。

## 回應格式

成功回應會把資源包在 `data` 裡：

```json
{ "data": { "id": "inv_..." } }
```

所有端點的錯誤回應都採用同一種結構：

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` 是穩定、供程式判讀的錯誤碼——分支判斷請依據它，而不是 `message`。
- `fields` 只在欄位層級的驗證錯誤時出現。
- 額外的業務資訊放在 `meta` 裡回傳，例如 `retry_after`、`reason_codes` 或 `min_amount`。

請求內容大小上限為 4KB；超過會回傳 `413 request_body_too_large`。

## 建立帳單

### `POST /v1/invoices`

建立一張帳單，並回傳其摘要與付款指示。

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| 欄位 | 說明 |
| --- | --- |
| `amount` | 必填。十進位字串，範圍 `0.01`–`1000000.00`，最多兩位小數。回應中會正規化（`12.34` → `12.3400`）。業務幣別固定為 `USD` 並在回應中回傳；它不是請求欄位。 |
| `reference_id` | 選填的呼叫端參照，在「專案 + 模式」內唯一，最長 200 字元。以完全相同的條件重試會回傳既有帳單與 `200 OK`；條件不同則回傳 `409 reference_id_conflict`。 |
| `description` | 選填、付款人可見的文字，最長 500 字元。 |
| `return_url` | 選填的 `http(s)` URL，付款完成後顯示為「返回商家」按鈕，最長 1000 字元。省略 → 快照專案當下的預設返回網址。明確傳 `null` 或 `""` → 不設返回網址。以 `reference_id` 重試時，省略的 `return_url` 不會拿來與既有帳單比對——若重試必須斷言特定值，請明確傳入。 |

成功回應（`201 Created`；冪等重用時回傳 `200 OK` 並帶 `meta.result: "reused"`）：

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

值得了解的語義：

- **資金流向無法透過此 API 改動。**請求無法設定收款位址或合約組態——它們在帳單建立時取自你已驗證的專案設定。帳單建立後即不可變更，只有付款與結算狀態會變化。
- **`payment_options` 列出買家可用的付款方式**，每個「網路 + 代幣」組合一筆，依付款人所見順序排列（USDT 在 USDC 之前，其後是各代幣經審定的網路順序）。`collection_method` 是判別欄位：
  - `evm_deposit`——每張帳單專屬的 EVM 收款位址。監控期限內任何金額為正的轉帳，都會依其金額入帳；`suggested_amount`（`max(0, amount_due − pending)`）只是建議值，不是比對要求。
  - `direct_exact`——Solana/TRON 的商家位址搭配一個精確金額。買家必須在單筆轉帳中送出恰好 `exact_amount`（`invoice_amount + matching_increment`）；該增量是用來歸屬付款的手段，絕不會計入帳單款項。
  - 只有 `status: "ready"` 的選項帶有上述可付款欄位；`unavailable` 的選項只帶共通欄位。辨識選項請用（`chain_namespace`、`chain_reference`、`token_address`）組合，絕不要靠陣列位置或顯示用中繼資料。
- **`checkout_status` 是面向付款人的狀態**，每次回應時即時推導：`paid`（權威狀態為 paid/settling/settled）、`confirming`（鏈上證據待確認）、`expired`（已超過 `monitoring_ends_at`）、`open`（至少有一個就緒的選項）或 `unavailable`。它絕不能作為履約依據——請使用 `invoice.paid` webhook。
- **`payment_revision`** 從 `0` 開始，已確認入帳的付款集合每變動一次就恰好遞增一次（每筆新的測試付款也算）。用它來丟棄在較新版本之後才送達的舊帳單快照或舊 webhook。
- **金額一律是十進位字串，絕不是浮點數。**已付與應付金額使用 18 位小數。`amount_due` 是 `max(amount − amount_paid, 0)`，`amount_overpaid` 是 `max(amount_paid − amount, 0)`——請直接讀這些欄位，不要自己對金額做減法。
- **invoq 會在帳單建立後監控鏈上 7 天**（`monitoring_ends_at`）。在該時點或之後到帳的轉帳只會被記錄、不會入帳；監控期間內的邊緣情況，則以商家後台的手動對帳作為營運端的最後手段。
- 測試帳單回傳 `monitoring_ends_at: null`、`payment_options: []` 和 `checkout_status: "unavailable"`——付款改由下方的 test-payments 端點模擬。
- 每個專案的速率限制：正式模式每分鐘 3,000 次、每天 100,000 次；測試模式每分鐘 300 次、每天 10,000 次。

錯誤碼：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`（帶 `amount_too_small` / `amount_too_large` 欄位碼與 `meta.min_amount` / `meta.max_amount`）、`409 reference_id_conflict`、`409 project_archived`、`409 no_payment_options_available`（帶排序後的 `meta.reason_codes`：`no_merchant_address`、`merchant_address_provisioning`、`below_rail_minimum`、`rail_unavailable`、`scanner_unavailable`、`scanner_capacity_exhausted`、`matching_capacity_exhausted`——其中 `merchant_address_provisioning` 是暫時性的，通常一兩分鐘內就會解除）、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

## 查詢帳單

### `GET /v1/invoices/{id}`

回傳公開的帳單摘要、付款人可見的付款狀態、專案品牌資訊與付款指示。**不需要 API 金鑰**——帳單 id 是用在付款連結 URL 裡、可分享且無法被猜中的公開 id，所以付款介面輪詢的就是這個端點（GET 的 CORS 允許任何來源）。

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

- `status` 是由已確認的付款與結算事件支撐的帳單權威狀態：`unpaid`、`partially_paid`、`paid`、`settling`、`settled` 或 `review_required`。
- `review_required` 表示帳單正在等待人工審核。它**不是**已付款狀態——即使 `amount_paid` 看起來足夠，也不要據此履約。
- `checkout_status` 是前文所述、推導出的面向付款人狀態；正式帳單有待確認的鏈上證據時會顯示 `confirming`，權威 `status` 則維持不變，直到轉帳確認為止。
- `transfers` 是面向付款人的收據紀錄：已確認、且已計入這張帳單的收款轉帳，讓結帳頁能逐筆顯示鏈上交易並連到區塊瀏覽器。每筆包含 `chain_namespace`、`chain_reference`、標準格式的 `transaction_id`、`event_index`、`amount`（以帳單幣別計，採用與 `amount_paid` 相同的 18 位小數 scale；`direct_exact` 的金額不含匹配增量）以及 `explorer_transaction_url`（或 `null`）。只有已確認的轉帳會出現——待確認的轉帳仍可能因鏈重組（reorg）被移除——且只保留金額最大的 20 筆，讓打到公開收款位址的粉塵轉帳無法擠掉真正的付款。此欄位一定存在：在有轉帳確認之前為 `[]`，測試帳單則永遠是 `[]`。
- 僅供呼叫端使用的 `reference_id` 在此不回傳；只回傳面向付款人的 `project` 品牌欄位。
- id 格式不合法或不存在，都回傳 `404 invoice_not_found`。

## 模擬測試付款

### `POST /v1/invoices/{id}/test-payments`

為**測試**帳單新增一筆模擬付款，並回傳更新後的付款狀態。僅限 `sk_test_...` 金鑰使用——你可以藉此把 unpaid → paid → webhook 的完整流程走一遍，完全不碰鏈。

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` 必填，必須大於零，整數最多 15 位、小數最多 4 位（`5`、`5.0`、`5.0000` 都正規化為 `5.0000`）。
- `reference_id` 選填，最長 200 字元，且在單張帳單內冪等：以相同的正規化金額重用會回傳 `200 OK` 並帶 `meta.result: "reused"`；金額不同則回傳 `409 test_payment_reference_conflict`。
- 允許部分付款、足額付款與超額付款：`0 < amount_paid < amount` 時為 `partially_paid`，`amount_paid >= amount` 後為 `paid`。首次轉入 `paid` 會觸發一條邏輯上的 `invoice.paid` webhook，且每筆建立的付款都會遞增 `payment_revision`。
- 每個專案的建立上限為每分鐘 300 筆、每天 10,000 筆。

錯誤碼：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`、`404 invoice_not_found`、`409 project_archived`、`409 test_mode_required`、`409 test_payment_reference_conflict`、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

## Webhook

在商家後台設定 webhook URL——測試與正式模式各有自己的 URL 和簽章金鑰。

### 事件

**`invoice.paid`**——在帳單轉入已付款狀態（`paid`、`settling` 或 `settled`）時送出：

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

**`invoice.payment_reversed`**——在原已付款的帳單重新跌破帳單金額時送出（例如鏈重組移除了一筆已入帳的轉帳）。事件內容的結構相同，帶有帳單當前的 `status`、`amount_paid`、更高的 `payment_revision`，以及 `fully_paid_at: null`。請將其視為使先前的履約信號失效，具體處理依你自己的業務策略。

- `review_required` 絕不會觸發 `invoice.paid`。只有在審核通過、進入已付款狀態後，才會建立該事件。
- 真實發生的 paid → reversed → paid 序列會依序投遞 `invoice.paid`、`invoice.payment_reversed`，然後是一條新的 `invoice.paid`，各自帶有該次變動後的 `payment_revision`。
- `reference_id` 和 `fully_paid_at` 可為 null 但一定存在；`return_url` 與付款指示則刻意不包含。請在伺服器端以帳單 id 加上 `reference_id` 對帳。

### 驗證簽章

每條送出的 webhook 都包含：

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` 是以秒計的 Unix 時間戳。`v1` 是對 `<t>.<raw_body>` 計算的小寫十六進位 HMAC-SHA256，金鑰為對應模式的 webhook 簽章金鑰。請在解析前用**原始請求內容**驗證——重新序列化 JSON 可能改變位元組、使簽章失效。請拒絕超出你重放容忍範圍的時間戳。（官方 SDK 都附有 `verifyWebhook` 輔助函式。）

### 投遞與重試

- 投遞的 POST 請求逾時時間為 10 秒。
- 所有非 2xx 回應——**包括重新導向和 4xx**——連同網路錯誤與逾時，都會以有上限的退避重試：1 分鐘、5 分鐘、30 分鐘、2 小時，每次帶最多 20% 的抖動，總計最多五次嘗試。
- 投遞保證**至少一次**（at-least-once），且順序可能錯亂：請依事件 `id` 去除重複、保留 `data.invoice.payment_revision` 最大的快照，並盡快回應 `2xx`（先確認收到，再做實際處理）。

## 上線

當[快速開始](#快速開始)的流程能在你的測試 webhook 上跑通後（本機開發可用 ngrok、cloudflared 之類的隧道）：

1. 在商家後台建立 `sk_live_` 金鑰。
2. 在商家後台設定正式 webhook URL。
3. 在伺服器設定裡換上這把金鑰。其他一切不變：同樣的端點、同樣的結構——正式帳單現在會帶真實的 `payment_options`。

測試帳單和測試付款永遠不會上鏈，也永遠不會被計為真實付款。

## 支援

- X：中文 [@invoqcn](https://x.com/invoqcn) · English [@invoqmoney](https://x.com/invoqmoney)
- Discord：https://discord.gg/V8cVrg4dET
- Telegram：https://t.me/invoqmoney
- 電子郵件：help@invoq.money
