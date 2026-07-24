# API REST de invoq

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · **Español** · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Este documento es una traducción del README en inglés; si algo difiere, vale la [versión en inglés](../README.md).

Pagos en stablecoins para desarrolladores independientes. Sin custodia — los fondos se liquidan directo en tu propia billetera.

Esta es la referencia de la API REST pública de invoq. Si usas alguno de los SDKs oficiales ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)), estos envuelven exactamente estos endpoints — este documento es el contrato que siguen.

- **URL base:** `https://api.invoq.money`
- **Checkout alojado:** `https://pay.invoq.money/<id de factura>`
- **Panel** (claves de API, billetera de cobro, webhooks): `https://app.invoq.money`

## Cómo funciona

1. **Crea una factura** desde tu servidor (`POST /v1/invoices`).
2. **Deja que el comprador la pague.** Lo más fácil: mándalo al checkout alojado en `https://pay.invoq.money/<id de factura>` — muestra el monto, la dirección y el código QR, está disponible en diez idiomas y no te exige ningún trabajo de UI. O integra el mismo checkout en tu propio sitio con [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). El comprador envía USDC o USDT desde cualquier billetera o exchange.
3. **Entérate cuando esté pagada.** invoq confirma la transferencia on-chain y envía un webhook `invoice.paid` a tu servidor; la liquidación va directo a tu propia billetera.

## Inicio rápido

Consigue una clave de prueba (`sk_test_...`) en el panel y crea tu primera factura:

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

Para una factura de producción, el `id` de la respuesta es todo lo que necesitas — `https://pay.invoq.money/<id>` es tu página de pago. Esta es una factura de prueba, que no trae instrucciones de pago on-chain, así que en su lugar simula el pago:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Si configuraste una URL de webhook de prueba en el panel, esta recibe un `invoice.paid` firmado. Ese es el ciclo completo — el resto de este documento es el detalle.

## Autenticación

Los endpoints de servidor usan una clave secreta de API creada en el panel:

```http
Authorization: Bearer sk_test_...
```

- Las claves `sk_test_...` crean **facturas de prueba**: los pagos son simulados, nada se mueve on-chain y los webhooks son reales (firmados y entregados igual que los de producción).
- Las claves `sk_live_...` crean **facturas de producción** con instrucciones de pago on-chain reales.

El modo de la factura siempre viene de la clave — el cuerpo de la solicitud nunca acepta un campo `mode`. Mantén las claves secretas en tu servidor; nunca las incluyas en código del cliente.

## Envoltura de respuesta

Las respuestas exitosas envuelven el recurso en `data`:

```json
{ "data": { "id": "inv_..." } }
```

Las respuestas de error comparten una misma forma en todos los endpoints:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` es un código de error estable y legible por máquina — basa tu lógica en él, no en `message`.
- `fields` solo está presente en errores de validación a nivel de campo.
- El contexto de negocio adicional se devuelve en `meta`, por ejemplo `retry_after`, `reason_codes` o `min_amount`.

Los cuerpos de solicitud están limitados a 4KB; los cuerpos que lo excedan devuelven `413 request_body_too_large`.

## Crear una factura

### `POST /v1/invoices`

Crea una factura y devuelve su resumen e instrucciones de pago.

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Campo | Notas |
| --- | --- |
| `amount` | Requerido. Cadena decimal, `0.01`–`1000000.00`, con hasta 2 decimales. Se normaliza en las respuestas (`12.34` → `12.3400`). La moneda de negocio está fija en `USD` y se devuelve en las respuestas; no es un campo de la solicitud. |
| `reference_id` | Referencia opcional del lado del llamador, única por proyecto + modo, máx. 200 caracteres. Reintentar con términos idénticos devuelve la factura existente con `200 OK`; términos distintos devuelven `409 reference_id_conflict`. |
| `description` | Texto opcional visible para el pagador, máx. 500 caracteres. |
| `return_url` | URL `http(s)` opcional que se muestra como botón de regreso al comercio después del pago, máx. 1000 caracteres. Omitida → se guarda una instantánea de la URL de regreso predeterminada del proyecto. `null` o `""` explícitos → sin URL de regreso. En reintentos por `reference_id`, una `return_url` omitida no se valida contra la factura existente — pásala explícitamente cuando el reintento deba asegurar un valor específico. |

Respuesta exitosa (`201 Created`; la reutilización idempotente devuelve `200 OK` con `meta.result: "reused"`):

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

Semántica que conviene conocer:

- **Los fondos no se pueden redirigir a través de esta API.** La solicitud no puede definir direcciones de destino ni configuración de contratos — eso viene de la configuración verificada de tu proyecto al momento de crear la factura. Después de creada, una factura es inmutable salvo por su estado de pago y liquidación.
- **`payment_options` lista las formas en que el comprador puede pagar**, una entrada por red + token, en el orden que ve el pagador (USDT antes que USDC, luego el orden de redes revisado de cada token). `collection_method` es el discriminador:
  - `evm_deposit` — una dirección de depósito EVM por factura. Cualquier transferencia positiva y a tiempo acredita por su monto; `suggested_amount` (`max(0, amount_due − pending)`) es una guía, no un requisito de coincidencia.
  - `direct_exact` — una dirección de comercio en Solana/TRON con un monto exacto. El comprador debe enviar exactamente `exact_amount` (`invoice_amount + matching_increment`) en una sola transferencia; el incremento es la forma de atribuir el pago y nunca es crédito de la factura.
  - Solo una opción con `status: "ready"` trae los campos pagables de arriba; una opción `unavailable` trae solo los campos comunes. Identifica una opción por (`chain_namespace`, `chain_reference`, `token_address`), nunca por posición en el arreglo ni por metadatos de presentación.
- **`checkout_status` es el estado de cara al pagador**, derivado en cada respuesta: `paid` (el estado canónico es paid/settling/settled), `confirming` (evidencia on-chain pendiente), `expired` (pasado `monitoring_ends_at`), `open` (al menos una opción lista) o `unavailable`. Nunca autoriza el procesamiento del pedido — usa el webhook `invoice.paid`.
- **`payment_revision`** empieza en `0` y se incrementa exactamente una vez cada vez que cambia el conjunto de pagos confirmados acreditados (también con cada nuevo pago de prueba). Úsalo para descartar una instantánea de la factura o un webhook más viejo entregado después de uno más nuevo.
- **Los montos son cadenas decimales, nunca floats.** Los montos pagados/adeudados usan 18 decimales. `amount_due` es `max(amount − amount_paid, 0)` y `amount_overpaid` es `max(amount_paid − amount, 0)` — lee esos campos en lugar de restar dinero por tu cuenta.
- **invoq vigila la cadena durante 7 días** después de la creación (`monitoring_ends_at`). Una transferencia que llega en ese instante o después queda registrada pero no acredita nada; la conciliación manual del panel es el respaldo del operador para los casos límite dentro de la ventana.
- Las facturas de prueba devuelven `monitoring_ends_at: null`, `payment_options: []` y `checkout_status: "unavailable"` — los pagos se simulan mediante el endpoint de pagos de prueba más abajo.
- Límites de tasa por proyecto: en producción 3,000/minuto y 100,000/día; en prueba 300/minuto y 10,000/día.

Códigos de error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (con los códigos de campo `amount_too_small` / `amount_too_large` y `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (con `meta.reason_codes` ordenados: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — `merchant_address_provisioning` es transitorio y suele resolverse en uno o dos minutos), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Leer una factura

### `GET /v1/invoices/{id}`

Devuelve el resumen público de la factura, el estado de pago visible para el pagador, la marca del proyecto y las instrucciones de pago. **No requiere clave de API** — los ids de factura son ids públicos compartibles e imposibles de adivinar, usados en las URLs de los enlaces de pago, así que este es el endpoint que sondean las UIs de pago (CORS permite cualquier origen para GET).

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

- `status` es el estado canónico de la factura, respaldado por eventos confirmados de pago y liquidación: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` o `review_required`.
- `review_required` significa que la factura está pendiente de revisión manual. **No** es un estado pagado — no proceses el pedido con él, aunque `amount_paid` parezca suficiente.
- `checkout_status` es el estado derivado de cara al pagador descrito arriba; las facturas de producción con evidencia on-chain pendiente muestran `confirming` mientras el `status` canónico se mantiene sin cambios hasta que la transferencia se confirme.
- `transfers` es el historial de recibos de cara al pagador: transferencias entrantes confirmadas que acreditaron esta factura, para que un checkout pueda mostrar cada transacción on-chain y enlazar a un explorador de bloques. Cada entrada trae `chain_namespace`, `chain_reference`, el `transaction_id` canónico, `event_index`, `amount` (en unidades de la moneda de la factura con la misma escala de 18 decimales que `amount_paid`; para `direct_exact` excluye el incremento de coincidencia) y `explorer_transaction_url` (o `null`). Solo aparecen transferencias confirmadas — una pendiente aún podría descartarse por un reorg — con un tope de las 20 mayores por monto, para que el polvo enviado a una dirección de depósito pública no desplace al pago real. Siempre presente: `[]` hasta que se confirma una transferencia, y siempre `[]` en las facturas de prueba.
- El `reference_id`, exclusivo del llamador, se omite aquí; solo se devuelven los campos de marca de `project` de cara al pagador.
- Tanto los ids mal formados como los desconocidos devuelven `404 invoice_not_found`.

## Simular un pago de prueba

### `POST /v1/invoices/{id}/test-payments`

Agrega un pago simulado a una factura de **prueba** y devuelve el estado de pago actualizado. Solo está disponible con claves `sk_test_...` — así recorres el ciclo completo unpaid → paid → webhook sin tocar ninguna cadena.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` es requerido, debe ser mayor que cero, con hasta 15 dígitos enteros y 4 decimales (`5`, `5.0`, `5.0000` se normalizan a `5.0000`).
- `reference_id` es opcional, máx. 200 caracteres, e idempotente por factura: reutilizarlo con el mismo monto normalizado devuelve `200 OK` con `meta.result: "reused"`; un monto distinto devuelve `409 test_payment_reference_conflict`.
- Se permiten pagos parciales, completos y en exceso: `partially_paid` mientras `0 < amount_paid < amount`, `paid` una vez que `amount_paid >= amount`. La primera transición a `paid` dispara un webhook `invoice.paid` lógico, y cada pago creado incrementa `payment_revision`.
- La creación está limitada a 300 por minuto y 10,000 por día por proyecto.

Códigos de error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhooks

Configura las URLs de webhook en el panel — los modos de prueba y de producción tienen cada uno su propia URL y su propio secreto de firma.

### Eventos

**`invoice.paid`** — se envía cuando una factura pasa a un estado pagado (`paid`, `settling` o `settled`):

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

**`invoice.payment_reversed`** — se envía cuando una factura antes pagada vuelve a caer por debajo de su monto (por ejemplo, un reorg de la cadena eliminó una transferencia acreditada). Misma forma de payload, con el `status` actual de la factura, `amount_paid`, un `payment_revision` mayor y `fully_paid_at: null`. Trátalo como una invalidación de la señal de procesamiento anterior, según tu propia política de negocio.

- `review_required` nunca dispara `invoice.paid`. El evento se crea solo después de que la revisión se resuelve en un estado pagado.
- Una verdadera secuencia pagada → revertida → pagada entrega `invoice.paid`, `invoice.payment_reversed` y luego un nuevo `invoice.paid`, cada uno con su `payment_revision` resultante.
- `reference_id` y `fully_paid_at` pueden ser null pero siempre están presentes; `return_url` y las instrucciones de pago están ausentes a propósito. Concilia del lado del servidor por id de factura más `reference_id`.

### Verificar las firmas

Cada webhook saliente incluye:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` es un timestamp Unix en segundos. `v1` es el HMAC-SHA256 en hex minúsculas de `<t>.<raw_body>`, usando como clave el secreto de webhook del modo correspondiente. Verifica contra el **cuerpo sin procesar de la solicitud** antes de parsearlo — reserializar el JSON puede cambiar los bytes e invalidar la firma. Rechaza los timestamps fuera de tu ventana de tolerancia a replays. (Los SDKs oficiales incluyen un helper `verifyWebhook`.)

### Entrega y reintentos

- Los POST de entrega expiran a los 10 segundos.
- Toda respuesta que no sea 2xx — **incluidos los redirects y los 4xx** — además de los errores de red y los tiempos de espera agotados, se reintenta con una espera acotada: 1 minuto, 5 minutos, 30 minutos y luego 2 horas, cada una con hasta 20% de jitter, hasta un máximo de cinco intentos en total.
- La entrega es **al menos una vez** y puede llegar fuera de orden: deduplica por el `id` del evento, quédate con la instantánea que tenga el mayor `data.invoice.payment_revision` y responde `2xx` rápido (haz el trabajo después de confirmar la recepción).

## Pasar a producción

Cuando el ciclo del [Inicio rápido](#inicio-rápido) funcione contra tu webhook de prueba (un túnel como ngrok o cloudflared sirve para el desarrollo local):

1. Crea una clave `sk_live_` en el panel.
2. Configura tu URL de webhook de producción en el panel.
3. Cambia la clave en la configuración de tu servidor. Nada más cambia: mismos endpoints, mismas formas — las facturas de producción ahora traen `payment_options` reales.

Las facturas y los pagos de prueba nunca tocan una cadena y nunca cuentan como pagos reales.

## Soporte

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Correo: help@invoq.money
