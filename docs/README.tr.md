# invoq REST API'si

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · **Türkçe** · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Bu belge İngilizce README'nin çevirisidir; bir fark olursa [İngilizce sürüm](../README.md) esas alınır.

Bağımsız geliştiriciler için stablecoin ödemeleri. Saklamasız — para doğrudan kendi cüzdanınıza geçer.

Burası invoq'un herkese açık REST API'sinin referansı. Resmî SDK'lar — [Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby) — tam olarak bu uç noktaları sarmalar.

- **Temel URL:** `https://api.invoq.money`
- **Barındırılan ödeme sayfası:** `https://pay.invoq.money/<fatura id>`
- **Panel** (API anahtarları, tahsilat cüzdanı, webhook'lar): `https://app.invoq.money`

## Nasıl çalışır

1. Sunucunuzdan **bir fatura oluşturun** (`POST /v1/invoices`).
2. **Alıcının ödemesine izin verin.** En kolayı: onu `https://pay.invoq.money/<fatura id>` adresine yönlendirin — sayfa tutarı, adresi ve QR kodu gösterir, on dil konuşur ve sizden hiç arayüz işi istemez. Ya da aynı ödeme sayfasını [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) ile kendi sitenize gömün. Alıcı herhangi bir cüzdandan veya borsadan USDT ya da USDC gönderir.
3. **Ödeme gelince haber alın.** invoq transferi zincir üstünde doğrular ve sunucunuza bir `invoice.paid` webhook'u gönderir. Para doğrudan kendi cüzdanınıza geçer.

## Hızlı başlangıç

Panelden bir test anahtarı (`sk_test_...`) alın ve ilk faturanızı oluşturun:

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

Canlı bir faturada yanıttaki `id` size yeter — `https://pay.invoq.money/<id>` sizin ödeme sayfanızdır. Yukarıdaki bir test faturası; zincir üstü ödeme talimatı taşımaz, o yüzden ödemeyi simüle edin:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Panelde bir test webhook URL'si ayarladıysanız, imzalı bir `invoice.paid` alır. Döngünün tamamı bu — belgenin geri kalanı ayrıntılar.

## Kimlik doğrulama

Sunucu uç noktaları, panelde oluşturulan gizli bir API anahtarı kullanır:

```http
Authorization: Bearer sk_test_...
```

- `sk_test_...` anahtarları **test faturaları** oluşturur: ödemeler simüledir, zincirde hiçbir şey hareket etmez ve webhook'lar gerçektir — tıpkı canlıdaki gibi imzalanır ve teslim edilir.
- `sk_live_...` anahtarları, gerçek zincir üstü ödeme talimatları taşıyan **canlı faturalar** oluşturur.

Mod her zaman anahtardan gelir; istek gövdesi asla `mode` alanı kabul etmez. Gizli anahtarları sunucunuzda tutun, asla istemci kodunda değil.

`GET /v1/invoices/{id}` hiçbir anahtar istemez. Fatura id'leri, ödeme bağlantılarında kullanılan tahmin edilemez herkese açık id'lerdir; bu yüzden ödeme arayüzleri onu doğrudan tarayıcıdan yoklayabilir — CORS, `GET` ve `HEAD` için her kaynağa izin verir.

## İstekler ve yanıtlar

Başarılı yanıtlar kaynağı `data` içine sarar:

```json
{ "data": { "id": "inv_..." } }
```

Hatalar tüm uç noktalarda tek bir biçimi paylaşır:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- Dallanmayı `code` üzerinden yapın. `message` insanlar içindir ve haber verilmeden değişebilir.
- `fields` yalnızca alan düzeyindeki doğrulama hatalarında görünür; `location` değeri `body`, `query` ya da `path` olur.
- Ek iş bağlamı `meta` içinde gelir: `retry_after`, `reason_codes`, `min_amount` gibi.

Bilinmesi iyi olan birkaç kural:

- **Doğrulama katıdır.** Tanınmayan bir gövde anahtarı veya sorgu parametresi, `fields[].code: "unknown_field"` ile birlikte `400 invalid_request` döndürür — isteğe önbellek kırıcı bir parametre eklemek isteğin tamamını başarısız kılar.
- **Tutarlar her zaman ondalık dizgidir, asla kayan noktalı sayı değil.** Fatura tutarları 4, ödenen ve kalan tutarlar 18 ondalık basamak taşır; `payment_options` içindeki token tutarları ise tam olarak `token_decimals` kadar.
- **`429 rate_limited` ipucunu gövdede taşır**, tam saniye cinsinden `meta.retry_after` olarak. `Retry-After` başlığı gönderilmez — `meta`'yı okuyun.
- İstek gövdesi 4KB ile sınırlıdır; fazlası `413 request_body_too_large` olur.
- Her JSON yanıtı `Cache-Control: no-store` taşır — ödeme durumu yoklanarak izlenir, dolayısıyla kimse size bayat bir fatura veremez.
- `GET /`, kimlik doğrulaması olmayan bir canlılık yoklamasıdır. `204 No Content` döndürür.

Fatura oluşturma proje başına sınırlıdır: canlı 3.000/dakika ve 100.000/gün, test 300/dakika ve 10.000/gün.

## Fatura oluşturma

### `POST /v1/invoices`

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Alan | Notlar |
| --- | --- |
| `amount` | Zorunlu. Ondalık dizgi, `0.01`–`1000000.00`, en fazla 2 ondalık basamak. Yanıtlarda normalleştirilir (`12.34` → `12.3400`). Para birimi `USD` olarak sabittir ve yanıtlarda döner; istek alanı değildir. |
| `reference_id` | İsteğe bağlı kendi referansınız; proje ve mod başına benzersiz, en fazla 200 karakter. Aynı koşullarla yeniden denemek mevcut faturayı `200 OK` ve `meta.result: "reused"` ile döndürür; farklı koşullar `409 reference_id_conflict` döndürür. |
| `description` | İsteğe bağlı, ödeyene görünen metin, en fazla 500 karakter. |
| `return_url` | İsteğe bağlı `http(s)` URL'si, en fazla 1000 karakter — ödemeden sonra gösterilen satıcıya dönüş düğmesi. Vermezseniz projenin varsayılanı faturaya kopyalanır; dönüş URL'si istemiyorsanız `null` veya `""` gönderin. `reference_id` ile yeniden denemede, verilmeyen bir `return_url` mevcut faturaya karşı doğrulanmaz; o yüzden yeniden denemenin belirli bir değeri dayatması gerekiyorsa açıkça gönderin. |

`201 Created`, ya da idempotent yeniden kullanımda `200 OK`:

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

- **Fonlar bu API üzerinden başka yöne çevrilemez.** İstek, ödemenin gideceği adresleri veya kontrat yapılandırmasını belirleyemez; bunlar doğrulanmış proje ayarlarınızdan gelir. Oluşturulduktan sonra fatura, ödeme ve tahsilat durumu dışında değişmezdir.
- Test faturaları `monitoring_ends_at: null`, `payment_options: []` ve `checkout_status: "unavailable"` döndürür. Onları [test-payments uç noktasıyla](#post-v1invoicesidtest-payments) ödeyin.

Hata kodları: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (alan kodu `invalid_format`, ya da `meta.min_amount` / `meta.max_amount` ile birlikte `amount_too_small` / `amount_too_large`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

`409 no_payment_options_available`, hiçbir ödeme seçeneğinin oluşturulamadığı anlamına gelir ve sıralanmış bir `meta.reason_codes` taşır: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted`. `merchant_address_provisioning` geçicidir — yeni bir Solana veya TRON adresi hâlâ etkinleştiriliyordur ve aynı istek genellikle saniyeler sonra başarılı olur.

## Fatura okuma

### `GET /v1/invoices/{id}`

Faturanın ödeyen tarafındaki görünümü: özet, ödeme durumu, projenin marka bilgileri, ödeme talimatları ve gelen transferlerin dökümü. API anahtarı gerekmez — ödeme arayüzlerinin yokladığı uç nokta budur.

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

Oluşturma yanıtıyla karşılaştırıldığında burada `amount_paid`, `project` ve `transfers` eklenir, size özel `reference_id` ise çıkarılır. `description`, `return_url`, `project.name` ve `project.logo_url` alanlarının hepsi `null` olabilir — bir ödeme sayfası hiçbiri yokken de düzgün görünmelidir.

`transfers`, ödeme sayfasının her zincir üstü işlemi gösterip bir gezginle bağlantılandırabilmesi için tutulan tahsilat dökümüdür. Her kayıt `chain_namespace`, `chain_reference`, `transaction_id`, `event_index`, `amount` (fatura para biriminde, `amount_paid` ile aynı 18 basamaklı ölçekte; `direct_exact` için eşleştirme artırımını içermez) ve `explorer_transaction_url` (yapılandırılmış gezgin yoksa `null`) taşır.

Yalnızca onaylanmış transferler görünür — bekleyen bir transfer bir zincir yeniden düzenlemesinde hâlâ düşebilir — ve tutara göre en büyük 20 tanesiyle sınırlıdır, büyükten küçüğe sıralanır; böylece toz, gerçek ödemeyi listeden düşüremez. Alan her zaman vardır: onaylanmış transfer olana kadar `[]`, test faturalarında ise her zaman `[]`.

Hem bozuk biçimli hem de bilinmeyen id'ler `404 invoice_not_found` döndürür.

## Alıcı nasıl öder

`payment_options`, bu faturanın ödenebileceği yolları listeler; ağ ve token başına bir girdi, ödeyene gösterilen sırayla: USDT, USDC'den önce; ardından her token'ın ağ sırası. Seçenekler, adresleri ve tutarlarıyla birlikte, fatura oluşturulurken sabitlenir — sonradan yapılandırdığınız bir tahsilat adresi veya hat, var olan bir faturayı yeniden yazmaz. Yalnızca her seçeneğin `status` alanı her yanıtta yeniden hesaplanır.

Bir seçeneği (`chain_namespace`, `chain_reference`, `token_address`) üçlüsüyle tanıyın. Asla dizideki konumuyla, asla `network_label`, `display_symbol`, `logo_url` veya `chain_logo_url` ile değil; bunlar görüntüleme meta verisidir.

Ayırt edici alan `collection_method`'dur.

**`evm_deposit`** — yalnızca bu faturaya ait bir yatırma adresi:

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

Zamanında gelen her pozitif transfer, tutarı kadar faturaya sayılır. `suggested_amount` bir eşleşme şartı değil, yönlendirmedir: `max(0, amount_due − pending)` değerinin hattın ondalık basamağına **yukarı** yuvarlanmış hâlidir, dolayısıyla `amount_due` değerini bir token birimine kadar aşabilir. İkisinin eşit olduğunu varsaymayın.

**`direct_exact`** — Solana veya TRON adresiniz ve kesin bir tutar:

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

Alıcı tek transferde tam olarak `exact_amount` (`invoice_amount + matching_increment`) göndermelidir. Artırım, ödemenin bu faturaya bağlanma yoludur; size ulaşır ama asla faturaya sayılmaz.

Bu kesin tutar her zaman faturanın tamamını karşıladığı için, faturanın onaylanmış ya da bekleyen herhangi bir ödemesi olur olmaz — başka bir hattan gelmiş olsa bile — `direct_exact` seçeneği `unavailable` hâline gelir. Aynı nedenle kısmen ödenmiş bir doğrudan fatura tamamlanamaz: kalan tutar için yeni bir fatura kesin.

Yukarıdaki ödemeye dönük alanları yalnızca `status: "ready"` olan seçenek taşır. `unavailable` bir seçenek yalnızca ortak alanları taşır: hizmet dışı kalmıştır (manuel inceleme, engellenmiş bir adres ya da hat, duraklatılmış bir zincir, dolmuş bir ödeme penceresi) ve artık alıcıya sunulmamalıdır.

## Ödeme durumu

İki durum alanı var ve farklı sorulara yanıt veriyorlar.

**`status`**, onaylanmış ödemeler ve tahsilatla desteklenen kanonik muhasebe durumudur: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` veya `review_required`. `paid`, `settling` ve `settled` üçü de alıcının ödediği anlamına gelir; yalnızca paranın cüzdanınıza ne kadar ilerlediği bakımından ayrılırlar. `review_required`, faturanın manuel incelemede olduğu anlamına gelir — bu **ödenmiş** bir durum değildir; `amount_paid` yeterli görünse bile siparişi işleme almayın.

**`checkout_status`**, ödeyene dönük durumdur; her yanıtta türetilir ve şu sırayla değerlendirilir:

| Değer | Anlamı |
| --- | --- |
| `paid` | `status` değeri `paid`, `settling` veya `settled` |
| `confirming` | Zincir üstü kanıt geldi, henüz onaylanmadı |
| `expired` | `monitoring_ends_at` aşıldı |
| `open` | En az bir ödeme seçeneği `ready` |
| `unavailable` | Geri kalan her şey: inceleme, engellenmiş hatlar, ödenmemiş test faturaları |

**`checkout_status` sipariş işlemeye asla yetki vermez.** `invoice.paid` webhook'unu kullanın.

**`payment_revision`**, `0`'dan başlar ve faturaya sayılan onaylı ödeme kümesi her değiştiğinde bir artar: yeni bir transfer, bir geri alma, yeni bir test ödemesi ya da sayılmış bir transferin zincir üstü zamanının düzeltilmesi. Tek başına tahsilat onu oynatmaz ve `status` değişmezken bile değişebilir. Daha yenisinden sonra gelen bir fatura anlık görüntüsünü veya webhook'u elemek için kullanın.

`amount_due`, `max(amount − amount_paid, 0)`; `amount_overpaid` ise `max(amount_paid − amount, 0)` değeridir. Tutarları kendiniz çıkarmak yerine bu alanları okuyun.

## Ödeme penceresi

`monitoring_ends_at`, faturanın oluşturulmasından bir gün sonrasıdır ve tek sınır odur. Bir transfer yalnızca kendi zincir üstü zamanı pencerenin içine düşüyorsa otomatik olarak faturaya sayılır — fatura var olmadan öncesinden hiçbir şey, `monitoring_ends_at` anında veya sonrasından da hiçbir şey. Sizin saatiniz, bizim gözlem anımız ve webhook'un varış zamanı bu kararın parçası değildir.

Geç gelen bir ödeme kaybolmaz. Faturaya kaydedilir ve panelde görünür; orada işlemi adıyla göstererek onu faturaya sayabilirsiniz — bunu sizin yerinize hiçbir otomatik süreç ileri süremez. Ne kadar vaktiniz olduğu hatta bağlıdır:

- **EVM** — süre yok. Yatırma adresi yalnızca bu faturaya aittir ve bir daha hiç verilmez; dolayısıyla oraya ulaşan bir transfer başka bir şeyi ödemiş olamaz.
- **Solana ve TRON** — oluşturulmasından itibaren 21 gün. Kesin tutar, `monitoring_ends_at` sonrasında 20 gün daha ayrılmış kalır; bundan sonrası daha yeni bir faturaya ait olabilir ve geç gelen bir transferin hangisini ödediğini kimse söyleyemez. Belirleyici olan transferin ne zaman geldiğidir, sizin onu ne zaman ele aldığınız değil.

Entegrasyonunuz için bir sonucu var: **`invoice.paid`, ödeme sayfası `expired` göstermeye başladıktan çok sonra da gelebilir** ve hiçbir alan bunu ayırt etmez. Ödeme sayfası süresi dolduğunda siparişi iptal ediyor veya yeniden satıyorsanız, faturanın hâlâ açık olduğunu varsaymak yerine `invoice.paid` ile kendi sipariş durumunuzu karşılaştırın — ve her hâlükârda idempotent işleyin.

## Webhook'lar

Webhook URL'lerini panelde ayarlayın. Test ve canlı modun her birinin kendi URL'si ve imzalama gizli anahtarı vardır.

### Olaylar

**`invoice.paid`** — fatura ödenmiş bir duruma ulaştı (`paid`, `settling` veya `settled`):

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

**`invoice.payment_reversed`** — daha önce ödenmiş bir fatura tekrar tutarının altına düştü; örneğin bir zincir yeniden düzenlemesi, sayılmış bir transferi kaldırdı. Aynı yük biçimi; faturanın güncel `status` değeri, daha yüksek bir `payment_revision` ve `fully_paid_at: null` ile. Kendi iş politikanıza göre, önceki ödeme sinyalini geçersiz kılan bir olay olarak ele alın.

- `review_required` asla `invoice.paid` tetiklemez. Eşik inceleme sırasında aşılırsa olay yalnızca bir kez, inceleme sonuçlandıktan sonra gönderilir.
- Gerçek bir ödendi → geri alındı → ödendi dizisi sırasıyla `invoice.paid`, `invoice.payment_reversed` ve yeni bir `invoice.paid` teslim eder; her biri kendi `payment_revision` değeriyle.
- `reference_id` ve `fully_paid_at` `null` olabilir ama her zaman bulunur. `return_url` ve ödeme talimatları bilinçli olarak yoktur — sunucu tarafında fatura id'si artı `reference_id` ile eşleştirin.

### İmzaları doğrulama

Her teslimat şunları taşır:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t`, saniye cinsinden bir Unix zaman damgasıdır. `v1` ise `<t>.<raw_body>` değerinin, ilgili modun webhook gizli anahtarıyla üretilmiş küçük harfli onaltılık HMAC-SHA256 imzasıdır. Ayrıştırmadan önce **ham istek gövdesine** karşı doğrulayın — JSON'u yeniden serileştirmek baytları değiştirip imzayı geçersiz kılabilir. Yeniden oynatma toleransı pencerenizin dışındaki zaman damgalarını reddedin. Resmî SDK'lar bir `verifyWebhook` yardımcısıyla gelir.

### Teslimat ve yeniden denemeler

- Teslimat istekleri 10 saniyede zaman aşımına uğrar.
- 2xx olmayan her yanıt — **yönlendirmeler ve 4xx dahil** — artı ağ hataları ve zaman aşımları sınırlı geri çekilmeyle yeniden denenir: 1 dakika, 5 dakika, 30 dakika, sonra 2 saat; her biri en fazla %20 sapmayla ve toplam beş denemeyle.
- Teslimat **en az bir kezdir ve sırasız gelebilir.** Olay `id`'sine göre yinelenenleri ayıklayın, en yüksek `payment_revision` değerine sahip anlık görüntüyü saklayın ve hızlıca `2xx` dönün — işi onayladıktan sonra yapın.

## Test modu ve canlıya geçiş

### `POST /v1/invoices/{id}/test-payments`

Bir **test** faturasına simüle edilmiş ödeme ekler ve güncel ödeme durumunu döndürür. Yalnızca `sk_test_...` anahtarlarıyla kullanılır — ödenmemiş → ödendi → webhook döngüsünün tamamını zincire dokunmadan böyle çalıştırırsınız.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` zorunludur ve sıfırdan büyük olmalıdır; en fazla 15 tam basamak ve 4 ondalık basamak (`5`, `5.0` ve `5.0000` hepsi `5.0000` olarak normalleşir).
- `reference_id` isteğe bağlıdır, en fazla 200 karakterdir ve fatura başına idempotenttir: aynı referans aynı normalleştirilmiş tutarla `200 OK` ve `meta.result: "reused"` döndürür; farklı bir tutar `409 test_payment_reference_conflict` döndürür.
- Kısmi, tam ve fazla ödemelerin hepsi kabul edilir: `0 < amount_paid < amount` iken `partially_paid`, `amount_paid >= amount` olunca `paid`. `paid` durumuna ilk geçiş bir `invoice.paid` gönderir ve her yeni ödeme `payment_revision` değerini artırır.
- Proje başına dakikada 300, günde 10.000 ile sınırlıdır.

Yanıt, oluşturma biçimindeki faturaya ek olarak `amount_paid` ve `fully_paid_at` alanlarını ve `meta.result` bilgisini içerir.

Hata kodları: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

### Canlıya geçiş

Döngü test webhook'unuza karşı çalıştığında — yerel geliştirme için ngrok veya cloudflared gibi bir tünel iş görür:

1. Panelde bir `sk_live_` anahtarı oluşturun.
2. Canlı webhook URL'nizi ayarlayın.
3. Sunucu yapılandırmanızdaki anahtarı değiştirin.

Başka hiçbir şey değişmez: aynı uç noktalar, aynı şekiller, ve canlı faturalar artık gerçek `payment_options` taşır. Test faturaları ve test ödemeleri asla zincire dokunmaz ve asla gerçek ödeme sayılmaz.

## Destek

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- E-posta: help@invoq.money
