# API REST da invoq

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · [Français](./README.fr.md) · **Português** · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Este documento é uma tradução do README em inglês; se algo divergir, vale a [versão em inglês](../README.md).

Pagamentos em stablecoin para desenvolvedores independentes. Sem custódia — o dinheiro cai direto na sua carteira.

Esta é a referência da API REST pública da invoq. Se você usa um dos SDKs oficiais ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)), eles encapsulam exatamente estes endpoints — este documento é o contrato que eles seguem.

- **URL base:** `https://api.invoq.money`
- **Checkout hospedado:** `https://pay.invoq.money/<id da fatura>`
- **Painel** (chaves de API, carteira de recebimento, webhooks): `https://app.invoq.money`

## Como funciona

1. **Crie uma fatura** a partir do seu servidor (`POST /v1/invoices`).
2. **Deixe o comprador pagá-la.** O caminho mais fácil: envie o comprador para o checkout hospedado em `https://pay.invoq.money/<id da fatura>` — ele mostra o valor, o endereço e o QR code, está disponível em dez idiomas e não exige nenhum trabalho de UI da sua parte. Ou incorpore o mesmo checkout no seu próprio site com [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). O comprador envia USDT ou USDC de qualquer carteira ou exchange.
3. **Fique sabendo quando ela for paga.** A invoq confirma a transferência on-chain e envia um webhook `invoice.paid` para o seu servidor; a liquidação vai direto para a sua própria carteira.

## Início rápido

Pegue uma chave de teste (`sk_test_...`) no painel e crie a sua primeira fatura:

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

Para uma fatura de produção, o `id` da resposta é tudo de que você precisa — `https://pay.invoq.money/<id>` é a sua página de checkout. Esta aqui é uma fatura de teste, que não carrega instruções de pagamento on-chain — então, em vez disso, simule o pagamento:

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Se você configurou uma URL de webhook de teste no painel, ela recebe um `invoice.paid` assinado. Esse é o fluxo completo — o resto deste documento é detalhe.

## Autenticação

Os endpoints de servidor usam uma chave de API secreta criada no painel:

```http
Authorization: Bearer sk_test_...
```

- Chaves `sk_test_...` criam **faturas de teste**: os pagamentos são simulados, nada se move on-chain e os webhooks são reais (assinados e entregues como os de produção).
- Chaves `sk_live_...` criam **faturas de produção** com instruções de pagamento on-chain reais.

O modo da fatura sempre vem da chave — o corpo da requisição nunca aceita um campo `mode`. Mantenha as chaves secretas no seu servidor; nunca as coloque em código que roda no cliente.

## Envelope de resposta

Respostas de sucesso envolvem o recurso em `data`:

```json
{ "data": { "id": "inv_..." } }
```

Respostas de erro compartilham um único formato em todos os endpoints:

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` é um código de erro estável e legível por máquina — decida a partir dele, não de `message`.
- `fields` só aparece em erros de validação de campo.
- Contexto de negócio extra vem em `meta`, como `retry_after`, `reason_codes` ou `min_amount`.

Os corpos de requisição são limitados a 4 KB; corpos maiores retornam `413 request_body_too_large`.

## Criar uma fatura

### `POST /v1/invoices`

Cria uma fatura e retorna seu resumo e as instruções de pagamento.

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Campo | Observações |
| --- | --- |
| `amount` | Obrigatório. String decimal, `0.01`–`1000000.00`, com até 2 casas decimais. Normalizado nas respostas (`12.34` → `12.3400`). A moeda de negócio é fixa em `USD` e retornada nas respostas; não é um campo da requisição. |
| `reference_id` | Referência opcional do seu lado, única por projeto + modo, máx. 200 caracteres. Repetir com termos idênticos retorna a fatura existente com `200 OK`; termos diferentes retornam `409 reference_id_conflict`. |
| `description` | Texto opcional visível ao pagador, máx. 500 caracteres. |
| `return_url` | URL `http(s)` opcional, exibida como o botão de retorno ao lojista depois do pagamento, máx. 1.000 caracteres. Omitida → a URL de retorno padrão do projeto é gravada na fatura (snapshot). `null` ou `""` explícitos → sem URL de retorno. Em repetições com `reference_id`, uma `return_url` omitida não é validada contra a fatura existente — passe-a explicitamente quando a repetição precisar garantir um valor específico. |

Resposta de sucesso (`201 Created`; o reuso idempotente retorna `200 OK` com `meta.result: "reused"`):

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

Semântica que vale a pena conhecer:

- **Fundos não podem ser redirecionados por esta API.** A requisição não pode definir endereços de recebimento nem configuração de contrato — eles vêm das configurações verificadas do seu projeto no momento em que a fatura é criada. Depois de criada, uma fatura é imutável, exceto pelo seu estado de pagamento e liquidação.
- **`payment_options` lista as maneiras como o comprador pode pagar**, uma entrada por rede + token, na ordem exibida ao pagador (USDT antes de USDC, depois a ordem de redes revisada de cada token). `collection_method` é o discriminador:
  - `evm_deposit` — um endereço de depósito EVM por fatura. Qualquer transferência positiva dentro do prazo credita pelo seu valor; `suggested_amount` (`max(0, amount_due − pending)`) é orientação, não uma exigência de correspondência.
  - `direct_exact` — um endereço do lojista em Solana/TRON com um valor exato. O comprador precisa enviar exatamente `exact_amount` (`invoice_amount + matching_increment`) em uma única transferência; o incremento é como o pagamento é atribuído e nunca vira crédito da fatura.
  - Só uma opção com `status: "ready"` carrega os campos de pagamento acima; uma opção `unavailable` carrega apenas os campos comuns. Identifique uma opção por (`chain_namespace`, `chain_reference`, `token_address`), nunca pela posição no array nem pelos metadados de exibição.
- **`checkout_status` é o estado voltado ao pagador**, derivado a cada resposta: `paid` (o status canônico é paid/settling/settled), `confirming` (evidência on-chain pendente), `expired` (passou de `monitoring_ends_at`), `open` (pelo menos uma opção pronta) ou `unavailable`. Ele nunca autoriza o processamento do pedido — use o webhook `invoice.paid`.
- **`payment_revision`** começa em `0` e incrementa exatamente uma vez sempre que o conjunto confirmado de pagamentos creditados muda (cada novo pagamento de teste também). Use-o para descartar um snapshot de fatura ou um webhook mais antigo entregue depois de um mais novo.
- **Valores são strings decimais, nunca floats.** Valores pagos/devidos usam 18 casas decimais. `amount_due` é `max(amount − amount_paid, 0)` e `amount_overpaid` é `max(amount_paid − amount, 0)` — leia esses campos em vez de subtrair dinheiro por conta própria.
- **A invoq monitora a blockchain por 7 dias** após a criação (`monitoring_ends_at`). Uma transferência que chega nesse instante ou depois dele é registrada, mas não credita nada; a reconciliação manual no painel é a rede de segurança do operador para casos-limite dentro da janela.
- Faturas de teste retornam `monitoring_ends_at: null`, `payment_options: []` e `checkout_status: "unavailable"` — os pagamentos são simulados pelo endpoint de test-payments abaixo.
- Limites de requisições por projeto: produção 3.000/minuto e 100.000/dia; teste 300/minuto e 10.000/dia.

Códigos de erro: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (com os códigos de campo `amount_too_small` / `amount_too_large` e `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (com `meta.reason_codes` ordenados: `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — `merchant_address_provisioning` é transitório e normalmente se resolve em um ou dois minutos), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Consultar uma fatura

### `GET /v1/invoices/{id}`

Retorna o resumo público da fatura, o estado de pagamento visível ao pagador, a identidade visual do projeto e as instruções de pagamento. **Não exige chave de API** — ids de fatura são ids públicos compartilháveis e impossíveis de adivinhar, usados nas URLs de link de pagamento, então este é o endpoint que as UIs de pagamento consultam via polling (o CORS permite qualquer origem para GET).

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

- `status` é o status canônico da fatura, sustentado por eventos confirmados de pagamento e liquidação: `unpaid`, `partially_paid`, `paid`, `settling`, `settled` ou `review_required`.
- `review_required` significa que a fatura está aguardando revisão manual. **Não** é um estado pago — não processe o pedido com base nele, mesmo que `amount_paid` pareça suficiente.
- `checkout_status` é o estado derivado voltado ao pagador descrito acima; faturas de produção com evidência on-chain pendente mostram `confirming` enquanto o `status` canônico permanece inalterado até a transferência confirmar.
- `transfers` é a trilha de recibos voltada ao pagador: as transferências recebidas confirmadas que creditaram esta fatura, para que um checkout possa mostrar cada transação on-chain e linkar um explorador de blocos. Cada entrada carrega `chain_namespace`, `chain_reference`, o `transaction_id` canônico, `event_index`, `amount` (em unidades da moeda da fatura, na mesma escala de 18 casas decimais de `amount_paid`; para `direct_exact`, exclui o incremento de correspondência) e `explorer_transaction_url` (ou `null`). Só transferências confirmadas aparecem — uma pendente ainda poderia ser descartada por um reorg — limitadas às 20 maiores por valor, para que a poeira (dust) enviada a um endereço de depósito público não abafe o pagamento real. Sempre presente: `[]` até uma transferência confirmar, e sempre `[]` em faturas de teste.
- O `reference_id`, visível só para quem chama a API, é omitido aqui; apenas os campos de identidade visual de `project` voltados ao pagador são retornados.
- Tanto ids malformados quanto ids desconhecidos retornam `404 invoice_not_found`.

## Simular um pagamento de teste

### `POST /v1/invoices/{id}/test-payments`

Adiciona um pagamento simulado a uma fatura de **teste** e retorna o estado de pagamento atualizado. Disponível apenas com chaves `sk_test_...` — é assim que você exercita o fluxo completo unpaid → paid → webhook sem tocar em nenhuma blockchain.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` é obrigatório, precisa ser maior que zero, com até 15 dígitos inteiros e 4 casas decimais (`5`, `5.0`, `5.0000` normalizam para `5.0000`).
- `reference_id` é opcional, máx. 200 caracteres e idempotente por fatura: reutilizá-lo com o mesmo valor normalizado retorna `200 OK` com `meta.result: "reused"`; um valor diferente retorna `409 test_payment_reference_conflict`.
- Pagamentos parciais, totais e a maior são permitidos: `partially_paid` enquanto `0 < amount_paid < amount`, `paid` quando `amount_paid >= amount`. A primeira transição para `paid` dispara um único webhook lógico `invoice.paid`, e cada pagamento criado incrementa `payment_revision`.
- A criação é limitada a 300 por minuto e 10.000 por dia por projeto.

Códigos de erro: `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhooks

Configure as URLs de webhook no painel — teste e produção têm, cada um, sua própria URL e seu próprio segredo de assinatura.

### Eventos

**`invoice.paid`** — enviado quando uma fatura transiciona para um estado pago (`paid`, `settling` ou `settled`):

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

**`invoice.payment_reversed`** — enviado quando uma fatura antes paga volta a ficar abaixo do seu valor (por exemplo, um reorg da blockchain removeu uma transferência creditada). Mesmo formato de payload, com o `status` atual da fatura, `amount_paid`, um `payment_revision` maior e `fully_paid_at: null`. Trate-o como a invalidação do sinal de processamento anterior, conforme a sua própria política de negócio.

- `review_required` nunca dispara `invoice.paid`. O evento só é criado depois que a revisão libera a fatura para um estado pago.
- Uma sequência real de pago → revertido → pago entrega `invoice.paid`, `invoice.payment_reversed` e depois um novo `invoice.paid`, cada um com o `payment_revision` resultante.
- `reference_id` e `fully_paid_at` podem ser nulos, mas estão sempre presentes; `return_url` e as instruções de pagamento estão ausentes de propósito. Reconcilie no servidor usando o id da fatura junto com o `reference_id`.

### Verificando assinaturas

Todo webhook enviado inclui:

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` é um timestamp Unix em segundos. `v1` é o HMAC-SHA256 em hex minúsculo de `<t>.<raw_body>`, usando como chave o segredo de webhook específico do modo. Verifique contra o **corpo bruto da requisição** antes de fazer o parse — reserializar o JSON pode mudar os bytes e invalidar a assinatura. Rejeite timestamps fora da sua janela de tolerância a replay. (Os SDKs oficiais trazem um helper `verifyWebhook`.)

### Entrega e novas tentativas

- Os POSTs de entrega expiram após 10 segundos.
- Toda resposta não 2xx — **inclusive redirecionamentos e 4xx** — além de erros de rede e timeouts, é reenviada com backoff limitado: 1 minuto, 5 minutos, 30 minutos e depois 2 horas, cada um com até 20% de jitter, num total de até cinco tentativas.
- A entrega é **at-least-once** e pode chegar fora de ordem: desduplique pelo `id` do evento, guarde o snapshot com o maior `data.invoice.payment_revision` e responda `2xx` rápido (faça o trabalho depois de confirmar o recebimento).

## Indo para produção

Quando o fluxo do [Início rápido](#início-rápido) funcionar contra o seu webhook de teste (um túnel como ngrok ou cloudflared resolve para o desenvolvimento local):

1. Crie uma chave `sk_live_` no painel.
2. Configure a URL do seu webhook de produção no painel.
3. Troque a chave na configuração do seu servidor. Nada mais muda: mesmos endpoints, mesmos formatos — as faturas de produção agora carregam `payment_options` reais.

Faturas de teste e pagamentos de teste nunca tocam uma blockchain e nunca contam como pagamentos reais.

## Suporte

- X: [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord: https://discord.gg/V8cVrg4dET
- Telegram: https://t.me/invoqmoney
- E-mail: help@invoq.money
