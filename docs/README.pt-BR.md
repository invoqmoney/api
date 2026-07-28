# API REST da invoq

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · **Português** · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Este documento é uma tradução do README em inglês; se algo divergir, vale a [versão em inglês](../README.md).

Pagamentos em stablecoin, integrados ao seu produto. Sem custódia — o dinheiro cai direto na sua carteira.

Esta é a referência da API REST pública da invoq. Os SDKs oficiais — [Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby) — encapsulam exatamente estes endpoints.

- **URL base:** `https://api.invoq.money`
- **Checkout hospedado:** `https://pay.invoq.money/<id da fatura>`
- **Painel** (chaves de API, carteira de recebimento, webhooks): `https://app.invoq.money`
- **OpenAPI 3.1:** `https://api.invoq.money/openapi.json` — este contrato, legível por máquina

**Programa com IA? Cole isto.**

```
Adicione pagamentos em stablecoin ao meu projeto com invoq. Comece no modo de teste. Leia a documentação antes de escrever código: https://invoq.money/llms.txt
```

## Como funciona

1. **Crie uma fatura** a partir do seu servidor (`POST /v1/invoices`).
2. **Deixe o comprador pagar.** O mais simples: mande ele para `https://pay.invoq.money/<id da fatura>` — a página mostra o valor, o endereço e o QR code, fala dez idiomas e não exige nenhuma UI da sua parte. Ou embuta o mesmo checkout com [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). O comprador envia USDT ou USDC de qualquer carteira ou exchange.
3. **Seja avisado quando for paga.** A invoq confirma a transferência on-chain e envia um webhook `invoice.paid` para o seu servidor. A liquidação vai direto para a sua própria carteira.

## Início rápido

Pegue uma chave de teste (`sk_test_...`) no painel e crie sua primeira fatura:

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

Numa fatura de produção, o `id` da resposta é tudo de que você precisa — `https://pay.invoq.money/<id>` é a sua página de pagamento. Esta é uma fatura de teste, que não traz instruções de pagamento on-chain, então simule o pagamento:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Se você configurou uma URL de webhook de teste no painel, ela recebe um `invoice.paid` assinado. Esse é o fluxo inteiro — o resto deste documento são os detalhes.

## Autenticação

Os endpoints de servidor usam uma chave de API secreta criada no painel:

```http
Authorization: Bearer sk_test_...
```

- Chaves `sk_test_...` criam **faturas de teste**: os pagamentos são simulados, nada se move on-chain e os webhooks são reais — assinados e entregues como os de produção.
- Chaves `sk_live_...` criam **faturas de produção**, com instruções de pagamento on-chain reais.

O modo vem sempre da chave; o corpo da requisição nunca aceita um campo `mode`. Mantenha as chaves secretas no seu servidor, nunca no código do cliente.

`GET /v1/invoices/{id}` não precisa de chave. Os id de fatura são id públicos impossíveis de adivinhar, usados nos links de pagamento, então interfaces de checkout podem consultá-lo direto do navegador — o CORS permite qualquer origem em `GET` e `HEAD`.

## Requisições e respostas

Respostas bem-sucedidas embrulham o recurso em `data`:

```json
{ "data": { "id": "inv_..." } }
```

Os erros compartilham um único formato em todos os endpoints:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "unknown_field", "field": "currency", "location": "body", "message": "Unrecognized key." }
  ]
}
```

- Ramifique pelo `code`. `message` é para humanos e pode mudar sem aviso.
- `fields` só aparece em erros de validação por campo; `location` é `body`, `query` ou `path`.
- Contexto de negócio extra vai em `meta`: `retry_after`, `reason_codes`, `min_amount` e afins.

Algumas convenções que vale conhecer:

- **A validação é estrita.** Uma chave de corpo ou um parâmetro de query não reconhecido dá `400 invalid_request` com `fields[].code: "unknown_field"` — anexar um parâmetro anticache faz a requisição inteira falhar.
- **Valores são strings decimais, nunca floats.** Valores de fatura têm 4 casas decimais, valores pagos e em aberto 18, e valores em token dentro de `payment_options` exatamente `token_decimals`.
- **`429 rate_limited` coloca a dica no corpo**, como `meta.retry_after` em segundos inteiros. Nenhum cabeçalho `Retry-After` é enviado: leia o `meta`.
- O corpo da requisição é limitado a 4KB; acima disso é `413 request_body_too_large`.
- Toda resposta JSON leva `Cache-Control: no-store` — o estado do pagamento é consultado por polling, então nada pode te entregar uma fatura desatualizada.
- `GET /` é uma sonda de disponibilidade sem autenticação. Retorna `204 No Content`.
- `GET /openapi.json` serve este contrato — os três endpoints e os dois webhooks — como OpenAPI 3.1. Ele é construído a partir dos mesmos schemas com que a API valida, então não pode descrever um servidor diferente. Use-o para gerar um cliente em uma linguagem sem SDK.

A criação de faturas tem limites por projeto: produção 3.000/minuto e 100.000/dia, teste 300/minuto e 10.000/dia.

## Criar uma fatura

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
| `amount` | Obrigatório. String decimal, `0.01`–`1000000.00`, com até 2 casas decimais. Normalizado nas respostas (`12.34` → `12.3400`). A moeda é fixa em `USD` e retornada nas respostas; não é um campo da requisição. |
| `reference_id` | Referência sua, opcional, única por projeto e modo, no máximo 200 caracteres. Repetir com termos idênticos devolve a fatura existente com `200 OK` e `meta.result: "reused"`; termos diferentes devolvem `409 reference_id_conflict`. |
| `description` | Texto opcional visível ao pagador, no máximo 500 caracteres. |
| `return_url` | URL `http(s)` opcional, no máximo 1000 caracteres — o botão de voltar à loja exibido após o pagamento. Se omitir, o padrão do projeto é congelado na fatura; passe `null` ou `""` para não ter URL de retorno. Numa repetição por `reference_id`, uma `return_url` omitida não é comparada com a fatura existente, então passe-a explicitamente quando a repetição precisar afirmar um valor. |

`201 Created`, ou `200 OK` no reuso idempotente:

> Os SDKs oficiais devolvem apenas o recurso e descartam `meta.result`, então por meio deles um create e um reuso idempotente são indistinguíveis — que é justamente para isso que serve o `reference_id`. Baseie sua contabilidade no `reference_id` que você enviou; chame o endpoint diretamente se precisar da distinção.

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

- **Fundos não podem ser redirecionados por esta API.** A requisição não pode definir endereços de recebimento nem configuração de contrato: eles vêm das configurações verificadas do seu projeto. Depois de criada, uma fatura é imutável, exceto pelo seu estado de pagamento e liquidação.
- Faturas de teste retornam `monitoring_ends_at: null`, `payment_options: []` e `checkout_status: "unavailable"`. Pague-as pelo [endpoint de test-payments](#post-v1invoicesidtest-payments).

Códigos de erro: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (código de campo `invalid_format`, ou `amount_too_small` / `amount_too_large` com `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

`409 no_payment_options_available` significa que nenhuma opção de pagamento pôde ser emitida, e traz `meta.reason_codes` ordenado: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted`. `merchant_address_provisioning` é transitório: um endereço Solana ou TRON novo ainda está sendo ativado, e a mesma requisição costuma passar segundos depois.

## Consultar uma fatura

### `GET /v1/invoices/{id}`

A visão da fatura do lado do pagador: resumo, estado do pagamento, identidade visual do projeto, instruções de pagamento e o histórico de transferências recebidas. Não exige chave de API — é este o endpoint que as interfaces de checkout consultam.

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

Comparado com a resposta de criação, isto acrescenta `amount_paid`, `project` e `transfers`, e omite o seu `reference_id`. `description`, `return_url`, `project.name` e `project.logo_url` podem ser `null` — um checkout precisa renderizar sem nenhum deles.

`transfers` é o histórico de comprovantes, para que um checkout possa mostrar cada transação on-chain e linkar para um explorador. Cada item traz `chain_namespace`, `chain_reference`, `transaction_id`, `event_index`, `amount` (na moeda da fatura, na mesma escala de 18 casas de `amount_paid`; em `direct_exact` exclui o incremento de correspondência) e `explorer_transaction_url` (`null` quando não há nenhum configurado).

Só aparecem transferências confirmadas — uma pendente ainda pode cair numa reorganização da chain — limitadas às 20 maiores por valor, da maior para a menor, para que poeira não empurre o pagamento de verdade para fora da lista. Sempre presente: `[]` até uma transferência confirmar, e sempre `[]` em faturas de teste.

Tanto id malformados quanto desconhecidos retornam `404 invoice_not_found`.

## Como o comprador paga

`payment_options` lista as formas de pagar esta fatura, uma entrada por rede e token, na ordem exibida ao pagador: USDT antes de USDC, depois a ordem de redes de cada token. As opções, com seus endereços e valores, são fixadas na criação da fatura — um endereço de recebimento ou uma via que você configurar depois não reescreve uma fatura existente. Só o `status` de cada opção é reavaliado a cada resposta.

Identifique uma opção por (`chain_namespace`, `chain_reference`, `token_address`). Nunca pela posição no array, e nunca por `network_label`, `display_symbol`, `logo_url` ou `chain_logo_url`, que são metadados de exibição.

`collection_method` é o discriminador.

**`evm_deposit`** — um endereço de depósito que pertence só a esta fatura:

```json
{ "deposit_address": "0x20c124f3919bb502c6126cda5bd6e5287859d5ca", "suggested_amount": "12.340000" }
```

Qualquer transferência positiva dentro do prazo credita a fatura pelo seu valor. `suggested_amount` é orientação, não exigência de correspondência: é `max(0, amount_due − pending)` arredondado **para cima** para `token_decimals`, ou para 6 dígitos quando a via carrega mais — alguém redigita esse número à mão — e então completado com zeros até `token_decimals`. Então pode exceder `amount_due` em até `0.000001`. Não assuma que os dois são iguais.

**`direct_exact`** — seu endereço Solana ou TRON mais um valor exato:

```json
{
  "recipient_address": "TJRabPrwbZy45sbavfcjinPJC18kjpRTv8",
  "invoice_amount": "12.340000",
  "matching_increment": "0.009999",
  "exact_amount": "12.349999"
}
```

O comprador precisa enviar exatamente `exact_amount` (`invoice_amount + matching_increment`) numa única transferência. O incremento é como o pagamento é atribuído a esta fatura: ele chega até você, mas nunca vira crédito da fatura.

Como esse valor exato sempre cobre a fatura inteira, uma opção `direct_exact` vira `unavailable` assim que a fatura tem qualquer pagamento confirmado ou pendente, inclusive um que chegou por outra via. Pelo mesmo motivo, uma fatura direta parcialmente paga não pode ser completada: emita uma nova fatura para o saldo.

Só uma opção com `status: "ready"` carrega os campos de pagamento acima. Uma `unavailable` carrega apenas os campos comuns: saiu de serviço (revisão manual, endereço ou via bloqueada, chain pausada, janela de pagamento encerrada) e não deveria mais ser oferecida ao comprador.

## Estado do pagamento

Dois campos de estado, respondendo perguntas diferentes.

**`status`** é o estado contábil canônico, sustentado por pagamentos confirmados e liquidação: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` ou `review_required`. `paid`, `settling` e `settled` significam todos que o comprador pagou — diferem apenas em quanto os fundos já avançaram até a sua carteira. `review_required` significa que a fatura está em revisão manual — **não** é um estado pago, então não processe o pedido mesmo que `amount_paid` pareça suficiente.

**`checkout_status`** é o estado voltado ao pagador, derivado a cada resposta e avaliado nesta ordem:

| Valor | Significado |
| --- | --- |
| `paid` | `status` é `paid`, `settling` ou `settled` |
| `confirming` | Chegou evidência on-chain, ainda não confirmada |
| `expired` | Passou de `monitoring_ends_at` |
| `open` | Pelo menos uma opção de pagamento está `ready` |
| `unavailable` | Todo o resto: revisão, vias bloqueadas, faturas de teste não pagas |

**`checkout_status` nunca autoriza o processamento do pedido.** Use o webhook `invoice.paid`.

**`payment_revision`** começa em `0` e incrementa uma vez sempre que o conjunto de pagamentos confirmados e creditados muda: uma nova transferência, um estorno, um novo pagamento de teste, ou a correção do horário on-chain de uma transferência creditada. A liquidação sozinha não o move, e ele pode mudar sem que `status` mude. Use-o para descartar um snapshot de fatura ou um webhook que chegou depois de um mais novo.

`amount_due` é `max(amount − amount_paid, 0)` e `amount_overpaid` é `max(amount_paid − amount, 0)`. Leia esses campos em vez de subtrair valores por conta própria.

## A janela de pagamento

`monitoring_ends_at` é um dia depois da criação da fatura, e é o único limite. Uma transferência só é creditada automaticamente se o horário on-chain dela cair dentro da janela: nada de antes de a fatura existir, nada em `monitoring_ends_at` ou depois. Seu relógio, o nosso momento de observação e a hora de chegada do webhook não entram na conta.

Um pagamento atrasado não se perde. Ele fica registrado na fatura e visível no painel, onde você pode creditá-lo apontando a transação — uma afirmação que nenhum processo automático faz no seu lugar. Quanto tempo você tem depende da via:

- **EVM** — sem prazo. O endereço de depósito pertence só a esta fatura e nunca é reemitido, então uma transferência que chega nele não pode ter pago outra coisa.
- **Solana e TRON** — 21 dias desde a criação. O valor exato fica reservado por mais 20 dias além de `monitoring_ends_at`; depois disso ele pode pertencer a uma fatura mais nova, e ninguém consegue dizer qual uma transferência atrasada pagou. O que conta é quando a transferência chegou, não quando você a resolve.

Uma consequência para a sua integração: **`invoice.paid` pode chegar muito depois de o checkout já mostrar `expired`**, e nenhum campo distingue esse caso. Se você cancela ou revende um pedido quando o checkout dele expira, concilie `invoice.paid` com o estado do seu próprio pedido em vez de supor que a fatura ainda está aberta — e processe de forma idempotente de qualquer jeito.

## Webhooks

Configure as URLs de webhook no painel. Teste e produção têm cada um sua URL e seu segredo de assinatura.

### Eventos

**`invoice.paid`** — a fatura chegou a um estado pago (`paid`, `settling` ou `settled`):

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

**`invoice.payment_reversed`** — uma fatura antes paga voltou a ficar abaixo do seu valor, por exemplo porque uma reorganização da chain removeu uma transferência creditada. Mesmo formato de payload, com o `status` atual da fatura, um `payment_revision` maior e `fully_paid_at: null`. Trate como invalidação do sinal de pagamento anterior, conforme a sua própria política de negócio.

- `review_required` nunca dispara `invoice.paid`. Se o limiar for cruzado durante a revisão, o evento é enviado uma única vez, depois que a revisão é resolvida.
- Uma sequência real paga → estornada → paga entrega `invoice.paid`, `invoice.payment_reversed` e depois um novo `invoice.paid`, cada um com o seu `payment_revision` resultante.
- `reference_id` e `fully_paid_at` podem ser `null`, mas estão sempre presentes. `return_url` e as instruções de pagamento ficam deliberadamente de fora: concilie no servidor pelo id da fatura mais o `reference_id`.

### Verificando assinaturas

Toda entrega traz:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` é um timestamp Unix em segundos. `v1` é o HMAC-SHA256 em hexadecimal minúsculo de `<t>.<raw_body>`, com o segredo de webhook daquele modo. Verifique contra o **corpo cru** da requisição antes de parsear: reserializar o JSON pode mudar os bytes e invalidar a assinatura. Rejeite timestamps fora da sua janela de tolerância a replay. Os SDKs oficiais trazem um helper `verifyWebhook`.

### Entrega e novas tentativas

- As entregas expiram em 10 segundos.
- Toda resposta não-2xx — **incluindo redirects e 4xx** — mais erros de rede e timeouts é repetida com backoff limitado: 1 minuto, 5 minutos, 30 minutos e 2 horas, cada um com até 20% de jitter, em cinco tentativas no total.
- A entrega é **pelo menos uma vez e pode chegar fora de ordem.** Deduplique pelo `id` do evento, fique com o snapshot de maior `payment_revision` e responda `2xx` rápido — faça o trabalho depois de confirmar o recebimento.

## Modo de teste e ir para produção

### `POST /v1/invoices/{id}/test-payments`

Adiciona um pagamento simulado a uma fatura de **teste** e devolve o estado de pagamento atualizado. Disponível só com chaves `sk_test_...` — é assim que você percorre todo o fluxo não paga → paga → webhook sem tocar em nenhuma chain.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` é obrigatório e maior que zero, com até 15 dígitos inteiros e 4 casas decimais (`5`, `5.0` e `5.0000` normalizam para `5.0000`).
- `reference_id` é opcional, no máximo 200 caracteres, e idempotente por fatura: a mesma referência com o mesmo valor normalizado devolve `200 OK` e `meta.result: "reused"`; com outro valor devolve `409 test_payment_reference_conflict`.
- Pagamentos parciais, completos e a maior são todos permitidos: `partially_paid` enquanto `0 < amount_paid < amount`, `paid` quando `amount_paid >= amount`. A primeira passagem para `paid` envia um `invoice.paid`, e cada novo pagamento incrementa `payment_revision`.
- Limitado a 300 por minuto e 10.000 por dia por projeto.

A resposta é a fatura no formato de criação, mais `amount_paid` e `fully_paid_at`, com `meta.result`.

Códigos de erro: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

### Indo para produção

Quando o fluxo funcionar contra o seu webhook de teste — um túnel como ngrok ou cloudflared resolve para o desenvolvimento local:

1. Crie uma chave `sk_live_` no painel.
2. Configure a URL do seu webhook de produção.
3. Troque a chave na configuração do seu servidor.

Nada mais muda: mesmos endpoints, mesmos formatos, e as faturas de produção agora carregam `payment_options` reais. Faturas de teste e pagamentos de teste nunca tocam uma blockchain e nunca contam como pagamentos reais.

## Suporte

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- E-mail: help@invoq.money
