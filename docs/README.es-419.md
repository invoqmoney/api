# API REST de invoq

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · **Español** · [Français](./README.fr.md) · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Este documento es una traducción del README en inglés; si algo difiere, vale la [versión en inglés](../README.md).

Pagos con stablecoins para desarrolladores independientes. Sin custodia: los fondos llegan directo a tu billetera.

Esta es la referencia de la API REST pública de invoq. Los SDK oficiales — [Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby) — envuelven exactamente estos endpoints.

- **URL base:** `https://api.invoq.money`
- **Checkout alojado:** `https://pay.invoq.money/<id de factura>`
- **Panel** (claves de API, billetera de cobro, webhooks): `https://app.invoq.money`
- **OpenAPI 3.1:** `https://api.invoq.money/openapi.json` — este contrato, legible por máquina

**¿Programas con IA? Pega esto.**

```
Agrega pagos con stablecoins a mi proyecto con invoq. Empieza en modo de prueba. Lee la documentación antes de escribir código: https://invoq.money/llms.txt
```

## Cómo funciona

1. **Crea una factura** desde tu servidor (`POST /v1/invoices`).
2. **Deja que el comprador la pague.** Lo más simple: mándalo a `https://pay.invoq.money/<id de factura>`, que muestra el monto, la dirección y el código QR, habla diez idiomas y no te exige nada de UI. O incrusta el mismo checkout con [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). El comprador envía USDT o USDC desde cualquier billetera o exchange.
3. **Entérate cuando esté pagada.** invoq confirma la transferencia on-chain y envía un webhook `invoice.paid` a tu servidor. La liquidación va directo a tu propia billetera.

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

En una factura de producción, el `id` de la respuesta es todo lo que necesitas: `https://pay.invoq.money/<id>` es tu página de pago. Esta es una factura de prueba, que no trae instrucciones de pago on-chain, así que simula el pago:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Si configuraste una URL de webhook de prueba en el panel, va a recibir un `invoice.paid` firmado. Ese es todo el ciclo; el resto de este documento son los detalles.

## Autenticación

Los endpoints de servidor usan una clave de API secreta creada en el panel:

```http
Authorization: Bearer sk_test_...
```

- Las claves `sk_test_...` crean **facturas de prueba**: los pagos son simulados, no se mueve nada on-chain y los webhooks son reales — firmados y entregados igual que los de producción.
- Las claves `sk_live_...` crean **facturas de producción** con instrucciones de pago on-chain reales.

El modo siempre viene de la clave; el cuerpo de la solicitud nunca acepta un campo `mode`. Guarda las claves secretas en tu servidor, nunca en el código del cliente.

`GET /v1/invoices/{id}` no necesita clave. Los id de factura son id públicos imposibles de adivinar que se usan en los enlaces de pago, así que las interfaces de checkout pueden consultarlo desde el navegador — CORS permite cualquier origen para `GET` y `HEAD`.

## Solicitudes y respuestas

Las respuestas exitosas envuelven el recurso en `data`:

```json
{ "data": { "id": "inv_..." } }
```

Los errores comparten una sola forma en todos los endpoints:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- Ramifica según `code`. `message` es para humanos y puede cambiar sin aviso.
- `fields` aparece solo en errores de validación a nivel de campo; `location` es `body`, `query` o `path`.
- El contexto de negocio adicional va en `meta`: `retry_after`, `reason_codes`, `min_amount`, etc.

Algunas convenciones que conviene saber:

- **La validación es estricta.** Una clave del cuerpo o un parámetro de consulta no reconocido devuelve `400 invalid_request` con `fields[].code: "unknown_field"`: agregar un parámetro anticaché hace fallar la solicitud entera.
- **Los montos son cadenas decimales, nunca floats.** Los montos de factura llevan 4 decimales, los montos pagados y pendientes 18, y los montos en tokens dentro de `payment_options` exactamente `token_decimals`.
- **`429 rate_limited` lleva su pista en el cuerpo**, como `meta.retry_after` en segundos enteros. No se envía ningún encabezado `Retry-After`: lee `meta`.
- El cuerpo de la solicitud está limitado a 4KB; más que eso es `413 request_body_too_large`.
- Toda respuesta JSON lleva `Cache-Control: no-store`: el estado del pago se consulta por polling, así que nadie puede entregarte una factura desactualizada.
- `GET /` es una sonda de disponibilidad sin autenticación. Devuelve `204 No Content`.
- `GET /openapi.json` sirve este contrato —los tres endpoints y los dos webhooks— como OpenAPI 3.1. Se construye a partir de los mismos esquemas con los que la API valida, así que no puede describir un servidor distinto. Úsalo para generar un cliente en un lenguaje sin SDK.

La creación de facturas tiene límites de tasa por proyecto: producción 3.000/minuto y 100.000/día, prueba 300/minuto y 10.000/día.

## Crear una factura

### `POST /v1/invoices`

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
| `amount` | Obligatorio. Cadena decimal, `0.01`–`1000000.00`, hasta 2 decimales. Se normaliza en las respuestas (`12.34` → `12.3400`). La moneda es fija en `USD` y se devuelve en las respuestas; no es un campo de la solicitud. |
| `reference_id` | Referencia propia opcional, única por proyecto y modo, máximo 200 caracteres. Reintentar con términos idénticos devuelve la factura existente con `200 OK` y `meta.result: "reused"`; con términos distintos devuelve `409 reference_id_conflict`. |
| `description` | Texto opcional visible para el pagador, máximo 500 caracteres. |
| `return_url` | URL `http(s)` opcional, máximo 1000 caracteres: el botón de volver al comercio que se muestra tras el pago. Si la omites se toma una instantánea del valor por defecto del proyecto; pasa `null` o `""` para no tener URL de retorno. En un reintento con `reference_id`, una `return_url` omitida no se compara con la factura existente, así que pásala explícitamente cuando el reintento deba afirmar un valor. |

`201 Created`, o `200 OK` en una reutilización idempotente:

> Los SDK oficiales devuelven solo el recurso y descartan `meta.result`, así que a través de ellos una creación y una reutilización idempotente son indistinguibles — que es justamente para lo que sirve `reference_id`. Basa tu contabilidad en el `reference_id` que enviaste; llama al endpoint directamente si necesitas la distinción.

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

- **Los fondos no se pueden redirigir a través de esta API.** La solicitud no puede definir direcciones de destino ni configuración de contratos: eso viene de la configuración verificada de tu proyecto. Después de creada, una factura es inmutable salvo por su estado de pago y liquidación.
- Las facturas de prueba devuelven `monitoring_ends_at: null`, `payment_options: []` y `checkout_status: "unavailable"`. Págalas con el [endpoint de test-payments](#post-v1invoicesidtest-payments).

Códigos de error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (código de campo `invalid_format`, o `amount_too_small` / `amount_too_large` con `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

`409 no_payment_options_available` significa que no se pudo emitir ninguna opción de pago, y trae `meta.reason_codes` ordenado: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted`. `merchant_address_provisioning` es transitorio: una dirección nueva de Solana o TRON todavía se está activando, y la misma solicitud suele funcionar segundos después.

## Leer una factura

### `GET /v1/invoices/{id}`

La vista de la factura del lado del pagador: resumen, estado del pago, marca del proyecto, instrucciones de pago y el historial de transferencias recibidas. No requiere clave de API — este es el endpoint que consultan las interfaces de checkout.

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

Comparado con la respuesta de creación, esto agrega `amount_paid`, `project` y `transfers`, y omite tu `reference_id`. `description`, `return_url`, `project.name` y `project.logo_url` pueden ser `null`: un checkout tiene que renderizar sin ninguno de ellos.

`transfers` es el historial de recibos, para que un checkout pueda mostrar cada transacción on-chain y enlazar a un explorador. Cada entrada trae `chain_namespace`, `chain_reference`, `transaction_id`, `event_index`, `amount` (en la moneda de la factura, con la misma escala de 18 decimales que `amount_paid`; en `direct_exact` excluye el incremento de coincidencia) y `explorer_transaction_url` (`null` si no hay ninguno configurado).

Solo aparecen transferencias confirmadas — una pendiente todavía podría caerse por una reorganización de la cadena — con un tope de las 20 mayores por monto, de mayor a menor, para que el polvo no desplace al pago real. Siempre presente: `[]` hasta que una transferencia se confirma, y siempre `[]` en facturas de prueba.

Tanto los id mal formados como los desconocidos devuelven `404 invoice_not_found`.

## Cómo paga el comprador

`payment_options` lista las formas de pagar esta factura, una entrada por red y token, en el orden que ve el pagador: USDT antes que USDC, y luego el orden de redes de cada token. Las opciones, con sus direcciones y montos, quedan fijas al crear la factura: una dirección de cobro o una vía que configures después no reescribe una factura existente. Solo el `status` de cada opción se reevalúa en cada respuesta.

Identifica una opción por (`chain_namespace`, `chain_reference`, `token_address`). Nunca por su posición en el arreglo, y nunca por `network_label`, `display_symbol`, `logo_url` o `chain_logo_url`, que son metadatos de presentación.

`collection_method` es el discriminador.

**`evm_deposit`**: una dirección de depósito que pertenece solo a esta factura.

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

Cualquier transferencia positiva y a tiempo acredita la factura por su monto. `suggested_amount` es una guía, no un requisito de coincidencia: es `max(0, amount_due − pending)` redondeado **hacia arriba** a `token_decimals`, o a 6 dígitos cuando la vía lleva más —alguien reescribe esta cifra a mano—, y luego se rellena con ceros hasta `token_decimals`. Así que puede superar a `amount_due` por hasta `0.000001`. No asumas que son iguales.

**`direct_exact`**: tu dirección de Solana o TRON más un monto exacto.

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

El comprador debe enviar exactamente `exact_amount` (`invoice_amount + matching_increment`) en una sola transferencia. El incremento es la forma de atribuir el pago a esta factura: te llega a ti, pero nunca es crédito de la factura.

Como ese monto exacto siempre cubre la factura entera, una opción `direct_exact` pasa a `unavailable` apenas la factura tiene algún pago confirmado o pendiente, incluso si llegó por otra vía. Por lo mismo, una factura directa pagada en parte no se puede completar: emite una factura nueva por el saldo.

Solo una opción con `status: "ready"` trae los campos pagables de arriba. Una `unavailable` trae solo los campos comunes: quedó fuera de servicio (revisión manual, una dirección o vía bloqueada, una cadena en pausa, una ventana de pago vencida) y no debería ofrecerse al comprador.

## Estado del pago

Dos campos de estado, y responden preguntas distintas.

**`status`** es el estado contable canónico, respaldado por pagos confirmados y liquidación: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` o `review_required`. `paid`, `settling` y `settled` significan todos que el comprador pagó: solo se diferencian en cuánto avanzaron los fondos hacia tu billetera. `review_required` significa que la factura está en revisión manual: **no** es un estado pagado, así que no proceses el pedido aunque `amount_paid` parezca suficiente.

**`checkout_status`** es el estado de cara al pagador, derivado en cada respuesta y evaluado en este orden:

| Valor | Significado |
| --- | --- |
| `paid` | `status` es `paid`, `settling` o `settled` |
| `confirming` | Llegó evidencia on-chain y aún no está confirmada |
| `expired` | Pasado `monitoring_ends_at` |
| `open` | Al menos una opción de pago está `ready` |
| `unavailable` | Todo lo demás: revisión, vías bloqueadas, facturas de prueba impagas |

**`checkout_status` nunca autoriza el procesamiento del pedido.** Usa el webhook `invoice.paid`.

**`payment_revision`** empieza en `0` y se incrementa una vez cada vez que cambia el conjunto de pagos confirmados y acreditados: una transferencia nueva, una reversión, un pago de prueba nuevo o una corrección del tiempo on-chain de una transferencia acreditada. La liquidación por sí sola no lo mueve, y puede cambiar mientras `status` no cambia. Úsalo para descartar una instantánea de la factura o un webhook que llegó después de uno más nuevo.

`amount_due` es `max(amount − amount_paid, 0)` y `amount_overpaid` es `max(amount_paid − amount, 0)`. Lee esos campos en vez de restar montos por tu cuenta.

## La ventana de pago

`monitoring_ends_at` es un día después de creada la factura, y es el único límite. Una transferencia se acredita automáticamente solo si su propio tiempo on-chain cae dentro de la ventana: nada de antes de que la factura existiera, nada en `monitoring_ends_at` o después. Tu reloj, nuestro momento de observación y la hora de llegada del webhook no cuentan.

Un pago tardío no se pierde. Queda registrado en la factura y visible en el panel, donde puedes acreditarlo nombrando la transacción — una afirmación que ningún proceso automático puede hacer por ti. Cuánto tiempo tienes depende de la vía:

- **EVM**: sin plazo. La dirección de depósito pertenece solo a esta factura y nunca se reutiliza, así que una transferencia que llegue ahí no puede haber pagado otra cosa.
- **Solana y TRON**: 21 días desde la creación. El monto exacto queda reservado 20 días más allá de `monitoring_ends_at`; después de eso puede pertenecer a una factura más nueva, y nadie puede decir cuál pagó una transferencia tardía. Lo que cuenta es cuándo llegó la transferencia, no cuándo la revisas.

Una consecuencia para tu integración: **`invoice.paid` puede llegar mucho después de que el checkout mostrara `expired`**, y ningún campo lo distingue. Si cancelas o revendes un pedido cuando su checkout vence, concilia `invoice.paid` contra el estado de tu propio pedido en vez de asumir que la factura sigue abierta, y procésalo de forma idempotente en cualquier caso.

## Webhooks

Configura las URL de webhook en el panel. Prueba y producción tienen cada una su URL y su secreto de firma.

### Eventos

**`invoice.paid`**: la factura llegó a un estado pagado (`paid`, `settling` o `settled`):

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

**`invoice.payment_reversed`**: una factura antes pagada volvió a caer por debajo de su monto, por ejemplo porque una reorganización de la cadena eliminó una transferencia acreditada. Misma forma de payload, con el `status` actual de la factura, un `payment_revision` mayor y `fully_paid_at: null`. Trátalo como la invalidación de la señal de pago anterior, según tu propia política de negocio.

- `review_required` nunca dispara `invoice.paid`. Si el umbral se cruza durante la revisión, el evento se envía una sola vez, después de que la revisión se resuelve.
- Una secuencia real pagada → revertida → pagada entrega `invoice.paid`, `invoice.payment_reversed` y luego un nuevo `invoice.paid`, cada uno con su `payment_revision` resultante.
- `reference_id` y `fully_paid_at` pueden ser `null`, pero siempre están presentes. `return_url` y las instrucciones de pago están deliberadamente ausentes: concilia del lado del servidor con el id de la factura más `reference_id`.

### Verificar las firmas

Cada entrega trae:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` es un timestamp Unix en segundos. `v1` es el HMAC-SHA256 en hexadecimal minúsculo de `<t>.<raw_body>`, con el secreto de webhook de ese modo. Verifica contra el **cuerpo crudo** de la solicitud antes de parsear: volver a serializar el JSON puede cambiar los bytes e invalidar la firma. Rechaza timestamps fuera de tu ventana de tolerancia de replay. Los SDK oficiales incluyen un helper `verifyWebhook`.

### Entrega y reintentos

- Las entregas expiran a los 10 segundos.
- Toda respuesta que no sea 2xx — **incluidos redirects y 4xx** — más errores de red y timeouts se reintenta con backoff acotado: 1 minuto, 5 minutos, 30 minutos y 2 horas, cada uno con hasta 20% de jitter, hasta cinco intentos en total.
- La entrega es **al menos una vez y puede llegar desordenada.** Deduplica por `id` de evento, quédate con la instantánea de mayor `payment_revision` y responde `2xx` rápido: haz el trabajo después de confirmar.

## Modo de prueba y pasar a producción

### `POST /v1/invoices/{id}/test-payments`

Agrega un pago simulado a una factura de **prueba** y devuelve su estado de pago actualizado. Solo disponible con claves `sk_test_...`: así recorres todo el ciclo impaga → pagada → webhook sin tocar una cadena.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` es obligatorio y mayor que cero, con hasta 15 dígitos enteros y 4 decimales (`5`, `5.0` y `5.0000` se normalizan a `5.0000`).
- `reference_id` es opcional, máximo 200 caracteres, e idempotente por factura: la misma referencia con el mismo monto normalizado devuelve `200 OK` y `meta.result: "reused"`; con otro monto devuelve `409 test_payment_reference_conflict`.
- Se permiten pagos parciales, completos y en exceso: `partially_paid` mientras `0 < amount_paid < amount`, `paid` cuando `amount_paid >= amount`. El primer cruce a `paid` envía un `invoice.paid`, y cada pago nuevo incrementa `payment_revision`.
- Limitado a 300 por minuto y 10.000 por día por proyecto.

La respuesta es la factura con la forma de creación, más `amount_paid` y `fully_paid_at`, con `meta.result`.

Códigos de error: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

### Pasar a producción

Cuando el ciclo funcione contra tu webhook de prueba — un túnel como ngrok o cloudflared sirve para el desarrollo local:

1. Crea una clave `sk_live_` en el panel.
2. Configura tu URL de webhook de producción.
3. Cambia la clave en la configuración de tu servidor.

Nada más cambia: mismos endpoints, mismas formas, y las facturas de producción ahora traen `payment_options` reales. Las facturas y los pagos de prueba nunca tocan una cadena y nunca cuentan como pagos reales.

## Soporte

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- Email: help@invoq.money
