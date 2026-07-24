# invoq REST API'si

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · **Türkçe** · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Bu belge İngilizce README'nin çevirisidir; bir fark olursa [İngilizce sürüm](../README.md) esas alınır.

Bağımsız geliştiriciler için stablecoin ödemeleri. invoq parayı asla tutmaz — fonlar doğrudan kendi cüzdanınıza iner.

Bu, invoq'un herkese açık REST API'sinin referansıdır. Resmî SDK'lardan birini ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)) kullanıyorsanız, hepsi tam olarak bu uç noktaları sarmalar — uydukları sözleşme bu belgedir.

- **Temel URL:** `https://api.invoq.money`
- **Barındırılan ödeme sayfası:** `https://pay.invoq.money/<fatura id>`
- **Panel** (API anahtarları, tahsilat cüzdanı, webhook'lar): `https://app.invoq.money`

## Nasıl çalışır

1. **Sunucunuzdan bir fatura oluşturun** (`POST /v1/invoices`).
2. **Alıcının ödemesini sağlayın.** En kolayı: alıcıyı `https://pay.invoq.money/<fatura id>` adresindeki barındırılan ödeme sayfasına gönderin — tutarı, adresi ve QR kodunu gösterir, on dili destekler ve sizden hiçbir arayüz işi istemez. Ya da aynı ödeme sayfasını [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js) ile kendi sitenize gömün. Alıcı, herhangi bir cüzdandan veya borsadan USDT ya da USDC gönderir.
3. **Ödendiğinde haberdar olun.** invoq transferi zincir üstünde doğrular ve sunucunuza bir `invoice.paid` webhook'u gönderir; para doğrudan kendi cüzdanınıza iner.

## Hızlı başlangıç

Panelden bir test anahtarı (`sk_test_...`) alın, ardından ilk faturanızı oluşturun:

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

Canlı bir faturada tek gereken yanıttaki `id`'dir — `https://pay.invoq.money/<id>` sizin ödeme sayfanızdır. Buradaki ise bir test faturası ve zincir üstü ödeme talimatı taşımaz; onun yerine ödemeyi simüle edin:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Panelde bir test webhook URL'si ayarladıysanız, oraya imzalı bir `invoice.paid` gelir. Döngünün tamamı bu — belgenin geri kalanı ayrıntıdan ibaret.

## Kimlik doğrulama

Sunucu uç noktaları, panelde oluşturulan gizli bir API anahtarı kullanır:

```http
Authorization: Bearer sk_test_...
```

- `sk_test_...` anahtarları **test faturaları** oluşturur: ödemeler simüle edilir, zincir üstünde hiçbir şey hareket etmez, webhook'lar ise gerçektir (canlıdakiler gibi imzalanır ve teslim edilir).
- `sk_live_...` anahtarları, gerçek zincir üstü ödeme talimatları taşıyan **canlı faturalar** oluşturur.

Fatura modu her zaman anahtardan gelir — istek gövdesi asla bir `mode` alanı kabul etmez. Gizli anahtarları sunucunuzda tutun; istemci koduyla asla dağıtmayın.

## Yanıt zarfı

Başarılı yanıtlar kaynağı `data` içine sarar:

```json
{ "data": { "id": "inv_..." } }
```

Hata yanıtları tüm uç noktalarda tek bir şekli paylaşır:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code`, kararlı ve makine tarafından okunabilir bir hata kodudur — dallanmayı `message`'a göre değil, buna göre yapın.
- `fields` yalnızca alan düzeyindeki doğrulama hatalarında bulunur.
- Ek iş bağlamı `meta` içinde döner: `retry_after`, `reason_codes` veya `min_amount` gibi.

İstek gövdeleri 4KB ile sınırlıdır; daha büyük gövdeler `413 request_body_too_large` döndürür.

## Fatura oluşturma

### `POST /v1/invoices`

Bir fatura oluşturur; özetini ve ödeme talimatlarını döndürür.

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
| `amount` | Zorunlu. `0.01`–`1000000.00` arası, en fazla 2 ondalık basamaklı ondalık dize. Yanıtlarda normalize edilir (`12.34` → `12.3400`). İş para birimi `USD` olarak sabittir ve yanıtlarda döner; bir istek alanı değildir. |
| `reference_id` | İsteğe bağlı, çağıran tarafın referansı; proje + mod başına benzersiz, en fazla 200 karakter. Aynı koşullarla yeniden denemek mevcut faturayı `200 OK` ile döndürür; farklı koşullar `409 reference_id_conflict` döndürür. |
| `description` | İsteğe bağlı, ödeyene görünen metin; en fazla 500 karakter. |
| `return_url` | İsteğe bağlı; ödemeden sonra satıcıya dönüş butonu olarak gösterilen `http(s)` URL'si, en fazla 1000 karakter. Verilmezse → projenin varsayılan dönüş URL'sinin anlık görüntüsü alınır. Açıkça `null` veya `""` → dönüş URL'si yok. `reference_id` ile yeniden denemelerde, verilmeyen `return_url` mevcut faturayla karşılaştırılıp doğrulanmaz — yeniden denemenin belirli bir değeri güvence altına alması gerekiyorsa açıkça geçirin. |

Başarılı yanıt (`201 Created`; idempotent yeniden kullanım, `meta.result: "reused"` ile `200 OK` döndürür):

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

Bilinmesi gerekenler:

- **Fonlar bu API üzerinden başka yöne çevrilemez.** İstek, ödemenin gideceği adresleri veya kontrat yapılandırmasını belirleyemez — bunlar, fatura oluşturulurken doğrulanmış proje ayarlarınızdan gelir. Oluşturulduktan sonra fatura, ödeme ve tahsilat durumu dışında değişmezdir.
- **`payment_options`, alıcının ödeyebileceği yolları listeler**; ağ + token başına bir girdi, ödeyene dönük sırayla (USDT, USDC'den önce; ardından her token'ın gözden geçirilmiş ağ sırası). Ayırt edici alan `collection_method`'dur:
  - `evm_deposit` — faturaya özel bir EVM yatırma adresi. Zamanında gelen her pozitif transfer, tutarı kadar faturaya sayılır; `suggested_amount` (`max(0, amount_due − pending)`) bir eşleşme şartı değil, yönlendirmedir.
  - `direct_exact` — kesin tutarlı bir Solana/TRON satıcı adresi. Alıcı, tek transferde tam olarak `exact_amount` (`invoice_amount + matching_increment`) göndermelidir; artırım, ödemenin hangi faturaya ait olduğunun belirlenme yoludur ve asla faturaya sayılmaz.
  - Yukarıdaki ödemeye dönük alanları yalnızca `status: "ready"` olan seçenek taşır; `unavailable` bir seçenek yalnızca ortak alanları taşır. Bir seçeneği (`chain_namespace`, `chain_reference`, `token_address`) üçlüsüyle tanıyın; asla dizideki konumuyla veya görüntüleme meta verisiyle değil.
- **`checkout_status`, ödeyene dönük durumdur** ve her yanıtta türetilir: `paid` (kanonik durum paid/settling/settled), `confirming` (zincir üstü kanıt beklemede), `expired` (`monitoring_ends_at` aşılmış), `open` (en az bir hazır seçenek var) veya `unavailable`. Sipariş işlemeye asla yetki vermez — `invoice.paid` webhook'unu kullanın.
- **`payment_revision`**, `0`'dan başlar ve faturaya sayılan onaylı ödeme kümesi her değiştiğinde (her yeni test ödemesinde de) tam olarak bir kez artar. Daha yenisinden sonra gelen eski bir fatura anlık görüntüsünü veya webhook'u elemek için kullanın.
- **Tutarlar ondalık dizedir, asla float değildir.** Ödenen/kalan tutarlar 18 ondalık basamak kullanır. `amount_due`, `max(amount − amount_paid, 0)`; `amount_overpaid` ise `max(amount_paid − amount, 0)` değeridir — parayı kendiniz çıkarmak yerine bu alanları okuyun.
- **invoq, oluşturmadan sonra zinciri 7 gün izler** (`monitoring_ends_at`). Tam o anda veya sonrasında gelen bir transfer kaydedilir ama faturaya sayılmaz; pencere içindeki uç durumlarda operatörün güvencesi, paneldeki manuel mutabakattır.
- Test faturaları `monitoring_ends_at: null`, `payment_options: []` ve `checkout_status: "unavailable"` döndürür — ödemeler, aşağıdaki test ödemeleri uç noktasıyla simüle edilir.
- Proje başına hız limitleri: canlıda dakikada 3.000 ve günde 100.000; testte dakikada 300 ve günde 10.000.

Hata kodları: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (`amount_too_small` / `amount_too_large` alan kodları ve `meta.min_amount` / `meta.max_amount` ile), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (sıralanmış `meta.reason_codes` ile: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — `merchant_address_provisioning` geçicidir ve genellikle bir iki dakika içinde düzelir), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Fatura okuma

### `GET /v1/invoices/{id}`

Herkese açık fatura özetini, ödeyene görünen ödeme durumunu, proje marka bilgisini ve ödeme talimatlarını döndürür. **API anahtarı gerekmez** — fatura id'leri, ödeme bağlantısı URL'lerinde kullanılan paylaşılabilir, tahmin edilemez herkese açık id'lerdir; dolayısıyla ödeme arayüzlerinin yokladığı uç nokta budur (CORS, GET için tüm origin'lere izin verir).

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

- `status`, onaylanmış ödeme ve tahsilat olaylarına dayanan kanonik fatura durumudur: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` veya `review_required`.
- `review_required`, faturanın manuel inceleme beklediği anlamına gelir. Bu bir ödendi durumu **değildir** — `amount_paid` yeterli görünse bile siparişi işlemeyin.
- `checkout_status`, yukarıda anlatılan türetilmiş, ödeyene dönük durumdur; zincir üstü kanıt bekleyen canlı faturalar `confirming` gösterir; kanonik `status` ise transfer onaylanana kadar değişmeden kalır.
- `transfers`, ödeyene dönük makbuz izidir: bu faturaya sayılan onaylanmış gelen transferler — böylece bir ödeme sayfası her zincir üstü işlemi gösterip blok gezginine bağlantı verebilir. Her girdi `chain_namespace`, `chain_reference`, kanonik `transaction_id`, `event_index`, `amount` (fatura para birimi cinsinden, `amount_paid` ile aynı 18 ondalık basamak ölçeğinde; `direct_exact` için eşleştirme artırımını içermez) ve `explorer_transaction_url` (veya `null`) taşır. Yalnızca onaylanmış transferler görünür — bekleyen bir transfer bir reorg ile hâlâ düşebilir — ve herkese açık bir yatırma adresine gönderilen toz (dust) gerçek ödemeyi gölgede bırakmasın diye tutara göre en büyük 20 girdiyle sınırlıdır. Her zaman mevcuttur: bir transfer onaylanana kadar `[]`, test faturalarında ise daima `[]`.
- Yalnızca çağırana özel olan `reference_id` burada yer almaz; `project` altında yalnızca ödeyene dönük marka alanları döner.
- Biçimi geçersiz id'ler de bilinmeyen id'ler de `404 invoice_not_found` döndürür.

## Test ödemesi simüle etme

### `POST /v1/invoices/{id}/test-payments`

Bir **test** faturasına simüle edilmiş bir ödeme ekler ve güncellenen ödeme durumunu döndürür. Yalnızca `sk_test_...` anahtarlarıyla kullanılabilir — zincire hiç dokunmadan unpaid → paid → webhook döngüsünün tamamını bu şekilde çalıştırırsınız.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` zorunludur ve sıfırdan büyük olmalıdır; en fazla 15 tam ve 4 ondalık basamak (`5`, `5.0` ve `5.0000`, `5.0000` olarak normalize edilir).
- `reference_id` isteğe bağlıdır, en fazla 200 karakterdir ve fatura başına idempotenttir: aynı normalize edilmiş tutarla yeniden kullanıldığında `meta.result: "reused"` ile `200 OK` döner; farklı bir tutar `409 test_payment_reference_conflict` döndürür.
- Kısmi, tam ve fazla ödemelere izin verilir: `0 < amount_paid < amount` iken `partially_paid`, `amount_paid >= amount` olduğunda `paid`. `paid` durumuna ilk geçiş, mantıksal olarak tek bir `invoice.paid` webhook'u tetikler; oluşturulan her ödeme `payment_revision`'ı artırır.
- Oluşturma, proje başına dakikada 300 ve günde 10.000 ile sınırlıdır.

Hata kodları: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhook'lar

Webhook URL'lerini panelde yapılandırın — test ve canlı modların her birinin kendi URL'si ve imzalama sırrı vardır.

### Olaylar

**`invoice.paid`** — bir fatura, ödeme tamamlanmış sayılan bir duruma (`paid`, `settling` veya `settled`) geçtiğinde gönderilir:

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

**`invoice.payment_reversed`** — daha önce ödenmiş bir fatura, tutarının altına geri düştüğünde gönderilir (örneğin bir zincir reorg'u, faturaya sayılmış bir transferi kaldırdığında). Payload şekli aynıdır; faturanın güncel `status` ve `amount_paid` değerleriyle, daha yüksek bir `payment_revision` ve `fully_paid_at: null` ile gelir. Bunu, kendi iş politikanız çerçevesinde, önceki sipariş işleme sinyalini geçersiz kılan bir olay olarak ele alın.

- `review_required` asla `invoice.paid` tetiklemez. Olay, ancak inceleme sonuçlanıp fatura ödeme tamamlanmış sayılan bir duruma geçtikten sonra oluşturulur.
- Gerçek bir ödendi → geri alındı → ödendi dizisi sırasıyla `invoice.paid`, `invoice.payment_reversed` ve yeni bir `invoice.paid` teslim eder; her biri sonuçtaki `payment_revision` değerini taşır.
- `reference_id` ve `fully_paid_at` null olabilir ama her zaman bulunur; `return_url` ve ödeme talimatları bilerek yoktur. Mutabakatı sunucu tarafında fatura id'si ve `reference_id` ile yapın.

### İmzaları doğrulama

Giden her webhook şunları içerir:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t`, saniye cinsinden bir Unix zaman damgasıdır. `v1`, moda özgü webhook sırrıyla anahtarlanmış `<t>.<raw_body>` değerinin küçük harfli onaltılık HMAC-SHA256'sıdır. Doğrulamayı ayrıştırmadan önce **ham istek gövdesi** üzerinde yapın — JSON'u yeniden serileştirmek baytları değiştirip imzayı geçersiz kılabilir. Tekrar oynatma (replay) tolerans pencerenizin dışındaki zaman damgalarını reddedin. (Resmî SDK'lar bir `verifyWebhook` yardımcısıyla gelir.)

### Teslimat ve yeniden denemeler

- Teslimat POST'ları 10 saniyede zaman aşımına uğrar.
- 2xx olmayan her yanıt — **yönlendirmeler ve 4xx dahil** — ayrıca ağ hataları ve zaman aşımları, sınırlı bir backoff ile yeniden denenir: 1 dakika, 5 dakika, 30 dakika, ardından 2 saat; her biri %20'ye varan jitter'la, toplamda en fazla beş deneme.
- Teslimat **en az bir kez** ilkesiyle çalışır ve sıralama bozulabilir: olay `id`'sine göre tekilleştirin, `data.invoice.payment_revision` değeri en büyük olan anlık görüntüyü tutun ve hızla `2xx` dönün (işi, aldığınızı onayladıktan sonra yapın).

## Canlıya geçiş

[Hızlı başlangıç](#hızlı-başlangıç) bölümündeki döngü test webhook'unuza karşı çalıştığında (yerel geliştirme için ngrok veya cloudflared gibi bir tünel iş görür):

1. Panelde bir `sk_live_` anahtarı oluşturun.
2. Canlı webhook URL'nizi panelde ayarlayın.
3. Sunucu yapılandırmanızdaki anahtarı değiştirin. Başka hiçbir şey değişmez: aynı uç noktalar, aynı şekiller — canlı faturalar artık gerçek `payment_options` taşır.

Test faturaları ve test ödemeleri asla zincire dokunmaz ve asla gerçek ödeme sayılmaz.

## Destek

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- E-posta: help@invoq.money
