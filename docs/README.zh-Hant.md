# invoq REST API

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · **繁體中文**

> 本文是英文版 README 的繁體中文翻譯；若表述有出入，以[英文版](../README.md)為準。

穩定幣收款，串接到你的產品。非託管——資金直接結算到你自己的錢包。

這是 invoq 公開 REST API 的參考文件。官方 SDK（[Node.js](https://github.com/invoqmoney/sdk-js)、[Python](https://github.com/invoqmoney/sdk-python)、[PHP](https://github.com/invoqmoney/sdk-php)、[Go](https://github.com/invoqmoney/sdk-go)、[Rust](https://github.com/invoqmoney/sdk-rust)、[Ruby](https://github.com/invoqmoney/sdk-ruby)）包裝的正是這些端點。

- **Base URL**：`https://api.invoq.money`
- **託管結帳頁**：`https://pay.invoq.money/<invoice id>`
- **商家後台**（API 金鑰、收款錢包、webhook）：`https://app.invoq.money`
- **OpenAPI 3.1**：`https://api.invoq.money/openapi.json` —— 同一份契約的機器可讀版

**在用 AI 寫程式？把這段貼給它。**

```
用 invoq 幫我的專案串接穩定幣收款，從測試模式開始。寫程式前先讀文件 https://invoq.money/llms.txt
```

## 運作方式

1. 在你的伺服器上**建立帳單**（`POST /v1/invoices`）。
2. **讓買家付款。**最簡單的做法：把買家導向 `https://pay.invoq.money/<invoice id>`——它會顯示金額、位址和 QR code，支援十種語言，你完全不用寫 UI。也可以用 [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) 把同一個結帳頁嵌進自己的網站。買家從任何錢包或交易所轉出 USDT 或 USDC 即可。
3. **付款完成時收到通知。**invoq 在鏈上確認轉帳後，會向你的伺服器送出 `invoice.paid` webhook；款項直接結算到你自己的錢包。

## 快速開始

先在商家後台拿一把測試金鑰（`sk_test_...`），然後建立你的第一張帳單：

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

正式帳單只需要回應中的 `id`——`https://pay.invoq.money/<id>` 就是你的結帳頁。上面這張是測試帳單，不帶鏈上付款指示，所以改用模擬付款：

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

如果你在商家後台設定了測試 webhook URL，它會收到一則帶簽章的 `invoice.paid`。整個流程到這裡就跑通了——本文件其餘部分都是細節。

## 身分驗證

伺服器端點使用在商家後台建立的私密 API 金鑰（secret key）：

```http
Authorization: Bearer sk_test_...
```

- `sk_test_...` 金鑰建立**測試帳單**：付款是模擬的，鏈上沒有任何資金移動，但 webhook 是真實的——簽章與投遞方式和正式環境完全相同。
- `sk_live_...` 金鑰建立**正式帳單**，帶有真實的鏈上付款指示。

模式一律由金鑰決定，請求內容不接受 `mode` 欄位。私密金鑰只放在你的伺服器上，絕不要放進用戶端程式碼。

`GET /v1/invoices/{id}` 不需要金鑰。帳單 id 是無法猜測的公開 id，會出現在付款連結中，所以結帳介面可以直接從瀏覽器輪詢它——`GET` 與 `HEAD` 的 CORS 允許任何來源。

## 請求與回應

成功的回應把資源包在 `data` 裡：

```json
{ "data": { "id": "inv_..." } }
```

所有端點的錯誤共用同一種結構：

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- 請判斷 `code`。`message` 是給人看的，隨時可能改動。
- `fields` 只在欄位層級的驗證錯誤時出現；`location` 為 `body`、`query` 或 `path`。
- 額外的業務資訊放在 `meta`，例如 `retry_after`、`reason_codes`、`min_amount`。

有幾項慣例值得先知道：

- **驗證是嚴格的。**無法識別的請求欄位或查詢參數會回傳 `400 invalid_request`，並帶 `fields[].code: "unknown_field"`——在請求上加一個防快取參數就會讓整個請求失敗。
- **金額一律是十進位字串，絕不是浮點數。**帳單金額 4 位小數，已付／未付金額 18 位小數，`payment_options` 裡的代幣金額則剛好等於 `token_decimals` 位。
- **`429 rate_limited` 把重試提示放在回應內容裡**，也就是 `meta.retry_after`，單位為整秒。不會送出 `Retry-After` 標頭——請讀 `meta`。
- 請求內容上限 4KB，超過會回傳 `413 request_body_too_large`。
- 每個 JSON 回應都帶 `Cache-Control: no-store`——付款狀態靠輪詢，任何環節都不該給你一份過期的帳單。
- `GET /` 是不需驗證的存活探測端點，回傳 `204 No Content`。
- `GET /openapi.json` 以 OpenAPI 3.1 提供這份契約：三個端點與兩個 webhook。它由 API 自己用來驗證的 schema 產生，因此不可能描述成另一台伺服器。沒有 SDK 的語言可以用它產生用戶端。

建立帳單的速率限制以專案計算：正式 3,000 次／分鐘、100,000 次／天，測試 300 次／分鐘、10,000 次／天。

## 建立帳單

### `POST /v1/invoices`

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
| `amount` | 必填。十進位字串，範圍 `0.01`–`1000000.00`，最多 2 位小數。回應中會正規化（`12.34` → `12.3400`）。幣別固定為 `USD` 並於回應中回傳，不是請求欄位。 |
| `reference_id` | 選填，你自己的參考編號，在同一專案 + 同一模式下唯一，最長 200 字元。以相同條件重試會回傳既有帳單，狀態碼 `200 OK`、`meta.result: "reused"`；條件不同則回傳 `409 reference_id_conflict`。 |
| `description` | 選填，買家可見的文字，最長 500 字元。 |
| `return_url` | 選填的 `http(s)` URL，最長 1000 字元——付款完成後顯示的返回商家按鈕。省略則沿用專案預設值的快照；傳 `null` 或 `""` 表示不要返回連結。以 `reference_id` 重試時，省略的 `return_url` 不會與既有帳單比對，所以當重試需要主張特定值時請明確傳入。 |

`201 Created`，冪等重用時為 `200 OK`：

> 官方 SDK 只回傳資源本身，會丟掉 `meta.result`，所以經過 SDK 時「新建」和「冪等重用」分不出來——這正是 `reference_id` 的用處。把帳目掛在你送出去的那個 `reference_id` 上；確實需要區分時，直接呼叫這個端點。

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

- **資金流向無法透過這個 API 改變。**請求不能設定收款位址或合約設定——它們來自你已驗證的專案設定。帳單一旦建立即不可變更，只有付款與結算狀態會變動。
- 測試帳單回傳 `monitoring_ends_at: null`、`payment_options: []` 和 `checkout_status: "unavailable"`，請用 [test-payments 端點](#post-v1invoicesidtest-payments)付款。

錯誤碼：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`（欄位碼為 `invalid_format`，或 `amount_too_small` / `amount_too_large` 並附 `meta.min_amount` / `meta.max_amount`）、`409 reference_id_conflict`、`409 project_archived`、`409 no_payment_options_available`、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

`409 no_payment_options_available` 表示沒能為這張帳單產生任何付款方式，並附排序後的 `meta.reason_codes`：`no_merchant_address`、`merchant_address_provisioning`、`below_rail_minimum`、`rail_unavailable`、`scanner_unavailable`、`scanner_capacity_exhausted`、`matching_capacity_exhausted`。其中 `merchant_address_provisioning` 是暫時性的——新增的 Solana 或 TRON 位址仍在啟用中，同樣的請求通常幾秒後就會成功。

## 查詢帳單

### `GET /v1/invoices/{id}`

帳單的買家視角：摘要、付款狀態、專案品牌資訊、付款指示，以及收款紀錄。不需要 API 金鑰——結帳介面輪詢的正是這個端點。

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

與建立回應相比，這裡多了 `amount_paid`、`project` 和 `transfers`，並且不回傳僅供呼叫方使用的 `reference_id`。`description`、`return_url`、`project.name`、`project.logo_url` 都可能是 `null`——結帳頁在它們全都缺少時也要能正常呈現。

`transfers` 是收款紀錄，結帳頁可以據此顯示每筆鏈上交易並連到區塊瀏覽器。每一筆帶有 `chain_namespace`、`chain_reference`、`transaction_id`、`event_index`、`amount`（帳單幣別，與 `amount_paid` 相同的 18 位小數；`direct_exact` 不含配對增額），以及 `explorer_transaction_url`（沒有設定瀏覽器時為 `null`）。

只有已確認的轉帳會出現（待確認的仍可能因鏈重組而消失），並取金額最大的 20 筆為上限、由大到小排列，這樣粉塵就無法把真正的付款擠出清單。這個欄位一定存在：在有轉帳確認之前是 `[]`，測試帳單則永遠是 `[]`。

格式錯誤的 id 與不存在的 id 都回傳 `404 invoice_not_found`。

## 買家如何付款

`payment_options` 列出這張帳單可用的付款方式，每個網路 + 代幣一筆，依買家端順序排列：USDT 在 USDC 之前，接著是各代幣的網路順序。有哪些選項、以及其中的位址與金額，在帳單建立時就固定下來——你之後設定的收款位址或通道不會改寫既有帳單。只有每個選項的 `status` 會在每次回應時重新計算。

辨識一個選項要用（`chain_namespace`、`chain_reference`、`token_address`）組合，絕不要靠陣列位置，也不要靠 `network_label`、`display_symbol`、`logo_url`、`chain_logo_url`——這些只是顯示用的中繼資料。

`collection_method` 是判別欄位。

**`evm_deposit`**——這張帳單專屬的收款位址：

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

任何在期限內到帳的正數轉帳都會依其金額入帳。`suggested_amount` 只是建議值，不是配對要求：它是 `max(0, amount_due − pending)` 依 `token_decimals` **無條件進位**，通道小數位超過 6 位時只取到 6 位（這串數字要人手抄），再補零回 `token_decimals`。因此它可能比 `amount_due` 多出最多 `0.000001`。不要主張兩者相等。

**`direct_exact`**——你的 Solana 或 TRON 位址搭配一個精確金額：

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

買家必須在單筆轉帳中剛好轉出 `exact_amount`（`invoice_amount + matching_increment`）。這個增額用來把付款歸屬到這張帳單；錢會進到你手上，但絕不會計入帳單金額。

正因為這個精確金額一律涵蓋整張帳單，只要帳單出現任何已確認或待確認的付款——即使來自另一條通道——`direct_exact` 選項就會變成 `unavailable`。同理，已部分付款的直轉帳單無法補足差額：請為餘額另開一張帳單。

只有 `status: "ready"` 的選項會帶上述可付款欄位。`unavailable` 的選項只帶共用欄位——它已停止服務（人工審核、位址或通道被封鎖、鏈暫停處理、付款時間窗已過），不該再提供給買家。

## 付款狀態

兩個狀態欄位，回答的是不同的問題。

**`status`** 是權威的記帳狀態，由已確認的付款與結算支撐：`unpaid`、`partially_paid`、`paid`、`settling`、`settled` 或 `review_required`。其中 `paid`、`settling`、`settled` 都表示買家已付款，差別只在於資金到你錢包的進度。`review_required` 表示帳單正在等待人工審核——它**不是**已付款狀態，即使 `amount_paid` 看起來足夠也不要出貨。

**`checkout_status`** 是面向買家的狀態，每次回應都重新推導，判定順序如下：

| 值 | 意義 |
| --- | --- |
| `paid` | `status` 為 `paid`、`settling` 或 `settled` |
| `confirming` | 鏈上證據已抵達，尚未確認 |
| `expired` | 已過 `monitoring_ends_at` |
| `open` | 至少有一個 `ready` 的付款選項 |
| `unavailable` | 其餘情況——審核中、通道被封鎖、未付款的測試帳單 |

**`checkout_status` 永遠不能當作出貨依據。**出貨請以 `invoice.paid` webhook 為準。

**`payment_revision`** 從 `0` 開始，每當已確認入帳的付款集合變動時剛好遞增一次：新增轉帳、發生沖銷、新增測試付款，或某筆已入帳轉帳的鏈上時間被修正。僅結算不會讓它變動，而且它可能在 `status` 不變時改變。用它來丟棄比新版本更晚抵達的帳單快照或 webhook。

`amount_due` 是 `max(amount − amount_paid, 0)`，`amount_overpaid` 是 `max(amount_paid − amount, 0)`。直接讀這兩個欄位，不要自己做金額減法。

## 付款時間窗

`monitoring_ends_at` 是帳單建立後的 1 天，也是唯一的邊界。只有鏈上時間落在窗內的轉帳才會被自動入帳——帳單存在之前的不算，剛好在 `monitoring_ends_at` 或之後的也不算。你的時鐘、我們的觀測時間、webhook 抵達時間都不參與判斷。

遲到的付款不會就此消失。它會被記錄在帳單上並顯示於商家後台，你可以在那裡指明這筆交易來為它入帳——這個主張是任何自動流程都無法代你做出的。你有多少時間取決於通道：

- **EVM**——沒有期限。收款位址只屬於這一張帳單，永不重複配發，因此轉入該位址的款項不可能屬於其他帳單。
- **Solana 與 TRON**——自建立起 21 天。精確金額會在 `monitoring_ends_at` 之後繼續保留 20 天；過了之後，它可能已經屬於更新的帳單，沒有人能斷定一筆遲到的轉帳付的是哪一張。判斷依據是轉帳落帳的時間，而不是你何時處理它。

這對你的整合有一個直接影響：**`invoice.paid` 可能在結帳頁顯示 `expired` 很久之後才抵達**，而且沒有任何欄位能區分它。如果你在結帳過期時就取消訂單或把商品再賣一次，請拿 `invoice.paid` 與你自己的訂單狀態對帳，而不要假設帳單還開著——無論如何都要冪等處理。

## Webhook

在商家後台設定 webhook URL。測試與正式各有自己的 URL 和簽章密鑰。

### 事件

**`invoice.paid`**——帳單進入已付款狀態（`paid`、`settling` 或 `settled`）：

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

**`invoice.payment_reversed`**——原本已付款的帳單又掉回金額之下，例如鏈重組移除了一筆已入帳的轉帳。內容結構相同，帶帳單目前的 `status`、更高的 `payment_revision` 和 `fully_paid_at: null`。請依你自己的營運政策，把它視為先前出貨訊號的失效。

- `review_required` 永遠不會觸發 `invoice.paid`。若門檻是在審核期間跨過的，事件會在審核通過後只送一次。
- 真實的 已付 → 沖銷 → 已付 序列會依序投遞 `invoice.paid`、`invoice.payment_reversed`，再一則 `invoice.paid`，各自帶上對應的 `payment_revision`。
- `reference_id` 與 `fully_paid_at` 可能為 `null`，但一定存在。`return_url` 與付款指示刻意不包含在內——請在伺服器端用帳單 id 加 `reference_id` 對帳。

### 驗證簽章

每次投遞都帶：

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` 是 Unix 時間戳（秒）。`v1` 是用該模式的 webhook 密鑰對 `<t>.<raw_body>` 做 HMAC-SHA256 後的小寫十六進位值。請在解析之前對**原始請求內容**驗章——把 JSON 重新序列化會改變位元組，讓簽章失效。同時拒絕超出你重播容忍範圍的時間戳。官方 SDK 都提供了這個驗章函式，各自按本語言的命名慣例（`verifyWebhook`、`verify_webhook`、`VerifyWebhook`）。

### 投遞與重試

- 投遞請求 10 秒逾時。
- 任何非 2xx 回應——**包括重新導向與 4xx**——以及網路錯誤和逾時都會重試，退避間隔為 1 分鐘、5 分鐘、30 分鐘、2 小時，每次帶最多 20% 抖動，總共 5 次。
- 投遞是**至少一次，且可能亂序。**請依事件 `id` 去重，保留 `payment_revision` 最大的那份快照，並盡快回 `2xx`——先回應，後處理。

## 測試模式與上線

### `POST /v1/invoices/{id}/test-payments`

為**測試**帳單加入一筆模擬付款，並回傳更新後的付款狀態。僅 `sk_test_...` 金鑰可用——這是你不碰鏈就跑完 未付款 → 已付款 → webhook 全流程的方式。

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` 必填且大於零，最多 15 位整數與 4 位小數（`5`、`5.0`、`5.0000` 都正規化為 `5.0000`）。
- `reference_id` 選填，最長 200 字元，在同一張帳單內冪等：相同編號 + 相同正規化金額回傳 `200 OK` 與 `meta.result: "reused"`，金額不同則回傳 `409 test_payment_reference_conflict`。
- 部分、全額與超額付款都允許：`0 < amount_paid < amount` 時為 `partially_paid`，`amount_paid >= amount` 時為 `paid`。首次跨入 `paid` 會送出一則 `invoice.paid`，每筆新付款都會讓 `payment_revision` 遞增。
- 限制為每專案 300 次／分鐘、10,000 次／天。

回應是建立端點的帳單結構，另加 `amount_paid` 與 `fully_paid_at`，並帶 `meta.result`。

錯誤碼：`401 invalid_secret_key`、`400 invalid_request`、`400 invalid_amount`、`404 invoice_not_found`、`409 project_archived`、`409 test_mode_required`、`409 test_payment_reference_conflict`、`413 request_body_too_large`、`429 rate_limited`、`500 server_misconfigured`。

### 上線

當整個流程在你的測試 webhook 上跑通之後（本地開發用 ngrok 或 cloudflared 這類通道就夠了）：

1. 在商家後台建立一把 `sk_live_` 金鑰。
2. 設定正式環境的 webhook URL。
3. 在伺服器設定裡換上這把金鑰。

其他什麼都不用改：端點一樣、結構一樣，只是正式帳單現在帶有真實的 `payment_options`。測試帳單與測試付款永遠不會碰到鏈，也永遠不會被計為真實收款。

## 支援

- X：[@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord：https://discord.gg/V8cVrg4dET
- Telegram：https://t.me/invoqmoney
- Email：help@invoq.money
