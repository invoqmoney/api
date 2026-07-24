# API REST invoq

[English](../README.md) · [Bahasa Indonesia](./README.id.md) · [Español](./README.es-419.md) · **Français** · [Português](./README.pt-BR.md) · [Tiếng Việt](./README.vi.md) · [Türkçe](./README.tr.md) · [ไทย](./README.th.md) · [简体中文](./README.zh-Hans.md) · [繁體中文](./README.zh-Hant.md)

> Ce document est une traduction du README anglais ; en cas de divergence, la [version anglaise](../README.md) fait foi.

Paiements en stablecoins pour développeurs indépendants. Non-custodial — les fonds arrivent directement dans votre propre portefeuille.

Ceci est la référence de l’API REST publique d’invoq. Si vous utilisez l’un des SDK officiels ([Node.js](https://github.com/invoqmoney/sdk-js), [Python](https://github.com/invoqmoney/sdk-python), [PHP](https://github.com/invoqmoney/sdk-php), [Go](https://github.com/invoqmoney/sdk-go), [Rust](https://github.com/invoqmoney/sdk-rust), [Ruby](https://github.com/invoqmoney/sdk-ruby)), ils encapsulent exactement ces endpoints — ce document est le contrat qu’ils suivent.

- **URL de base :** `https://api.invoq.money`
- **Page de paiement hébergée :** `https://pay.invoq.money/<id de facture>`
- **Tableau de bord** (clés API, portefeuille de réception, webhooks) : `https://app.invoq.money`

## Comment ça marche

1. **Créez une facture** depuis votre serveur (`POST /v1/invoices`).
2. **Laissez l’acheteur la payer.** Le plus simple : envoyez-le vers la page de paiement hébergée sur `https://pay.invoq.money/<id de facture>` — elle affiche le montant, l’adresse et le QR code, parle dix langues et ne vous demande aucun travail d’interface. Ou intégrez cette même page de paiement à votre propre site avec [`@invoq/checkout`](https://github.com/invoqmoney/sdk-js). L’acheteur envoie des USDT ou des USDC depuis n’importe quel portefeuille ou exchange.
3. **Soyez prévenu du paiement.** invoq confirme le transfert on-chain et envoie un webhook `invoice.paid` à votre serveur ; le règlement arrive directement dans votre propre portefeuille.

## Démarrage rapide

Récupérez une clé de test (`sk_test_...`) dans le tableau de bord, puis créez votre première facture :

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

Pour une facture de production, l’`id` de la réponse suffit — `https://pay.invoq.money/<id>` est votre page de paiement. Celle-ci est une facture de test, qui ne porte aucune instruction de paiement on-chain — simulez donc le paiement à la place :

```bash
curl https://api.invoq.money/v1/invoices/<id>/test-payments \
  -H "Authorization: Bearer sk_test_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": "12.34" }'
```

Si vous avez configuré une URL de webhook de test dans le tableau de bord, elle reçoit un `invoice.paid` signé. Voilà toute la boucle — le reste de ce document en est le détail.

## Authentification

Les endpoints serveur utilisent une clé API secrète créée dans le tableau de bord :

```http
Authorization: Bearer sk_test_...
```

- Les clés `sk_test_...` créent des **factures de test** : les paiements sont simulés, rien ne bouge on-chain, et les webhooks sont réels (signés et livrés comme ceux de production).
- Les clés `sk_live_...` créent des **factures de production** avec de vraies instructions de paiement on-chain.

Le mode de la facture vient toujours de la clé — le corps de la requête n’accepte jamais de champ `mode`. Gardez les clés secrètes sur votre serveur ; ne les embarquez jamais dans du code client.

## Enveloppe de réponse

Les réponses réussies enveloppent la ressource dans `data` :

```json
{ "data": { "id": "inv_..." } }
```

Les réponses d’erreur partagent la même forme sur tous les endpoints :

```json
{
  "code": "invalid_request",
  "message": "Invalid request.",
  "fields": [
    { "code": "invalid_number", "field": "page", "location": "query", "message": "Must be a number." }
  ]
}
```

- `code` est un code d’erreur stable, lisible par machine — branchez votre logique dessus, pas sur `message`.
- `fields` n’est présent que pour les erreurs de validation au niveau des champs.
- Le contexte métier supplémentaire est renvoyé dans `meta`, par exemple `retry_after`, `reason_codes` ou `min_amount`.

Les corps de requête sont limités à 4 Ko ; les corps trop volumineux renvoient `413 request_body_too_large`.

## Créer une facture

### `POST /v1/invoices`

Crée une facture et renvoie son résumé et ses instructions de paiement.

```json
{
  "amount": "12.34",
  "reference_id": "order_10086",
  "description": "Website audit for June",
  "return_url": "https://example.com/orders/order_10086"
}
```

| Champ | Notes |
| --- | --- |
| `amount` | Requis. Chaîne décimale, `0.01`–`1000000.00`, au plus 2 décimales. Normalisé dans les réponses (`12.34` → `12.3400`). La devise métier est fixée à `USD` et renvoyée dans les réponses ; ce n’est pas un champ de requête. |
| `reference_id` | Référence optionnelle côté appelant, unique par projet + mode, 200 caractères max. Réessayer avec des conditions identiques renvoie la facture existante avec `200 OK` ; des conditions différentes renvoient `409 reference_id_conflict`. |
| `description` | Texte optionnel visible par le payeur, 500 caractères max. |
| `return_url` | URL `http(s)` optionnelle affichée comme bouton de retour vers le marchand après paiement, 1000 caractères max. Omise → l’URL de retour par défaut du projet est figée dans la facture. `null` ou `""` explicite → pas d’URL de retour. Lors d’un réessai par `reference_id`, une `return_url` omise n’est pas validée contre la facture existante — passez-la explicitement quand le réessai doit garantir une valeur précise. |

Réponse réussie (`201 Created` ; une réutilisation idempotente renvoie `200 OK` avec `meta.result: "reused"`) :

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

Sémantique à connaître :

- **Les fonds ne peuvent pas être détournés via cette API.** La requête ne peut définir ni adresses de destinataire ni configuration de contrat — elles proviennent des réglages vérifiés de votre projet à la création de la facture. Après création, une facture est immuable, à l’exception de son état de paiement et de règlement.
- **`payment_options` liste les façons dont l’acheteur peut payer**, une entrée par réseau + token, dans l’ordre présenté au payeur (USDT avant USDC, puis l’ordre de réseaux validé de chaque token). `collection_method` est le discriminant :
  - `evm_deposit` — une adresse de dépôt EVM propre à la facture. Tout transfert positif arrivé dans les temps crédite son montant ; `suggested_amount` (`max(0, amount_due − pending)`) est une indication, pas une exigence de correspondance.
  - `direct_exact` — une adresse marchande Solana/TRON avec un montant exact. L’acheteur doit envoyer exactement `exact_amount` (`invoice_amount + matching_increment`) en un seul transfert ; l’incrément sert à attribuer le paiement et n’est jamais un crédit de la facture.
  - Seule une option `status: "ready"` porte les champs payables ci-dessus ; une option `unavailable` ne porte que les champs communs. Identifiez une option par (`chain_namespace`, `chain_reference`, `token_address`), jamais par sa position dans le tableau ni par ses métadonnées d’affichage.
- **`checkout_status` est l’état côté payeur**, dérivé à chaque réponse : `paid` (le statut canonique est paid/settling/settled), `confirming` (preuve on-chain en attente), `expired` (au-delà de `monitoring_ends_at`), `open` (au moins une option prête) ou `unavailable`. Il n’autorise jamais le traitement de la commande — utilisez le webhook `invoice.paid`.
- **`payment_revision`** démarre à `0` et s’incrémente exactement une fois à chaque changement de l’ensemble des paiements crédités confirmés (chaque nouveau paiement de test aussi). Utilisez-le pour écarter un instantané de facture ou un webhook plus ancien livré après un plus récent.
- **Les montants sont des chaînes décimales, jamais des flottants.** Les montants payés/dus utilisent 18 décimales. `amount_due` vaut `max(amount − amount_paid, 0)` et `amount_overpaid` vaut `max(amount_paid − amount, 0)` — lisez ces champs plutôt que de soustraire de l’argent vous-même.
- **invoq surveille la chaîne pendant 7 jours** après la création (`monitoring_ends_at`). Un transfert qui arrive à cet instant ou après est enregistré mais ne crédite rien ; la réconciliation manuelle du tableau de bord est le filet de sécurité de l’opérateur pour les cas limites à l’intérieur de la fenêtre.
- Les factures de test renvoient `monitoring_ends_at: null`, `payment_options: []` et `checkout_status: "unavailable"` — les paiements se simulent via l’endpoint de paiements de test ci-dessous.
- Limites de débit par projet : en production 3 000/minute et 100 000/jour ; en test 300/minute et 10 000/jour.

Codes d’erreur : `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount` (avec les codes de champ `amount_too_small` / `amount_too_large` et `meta.min_amount` / `meta.max_amount`), `409 reference_id_conflict`, `409 project_archived`, `409 no_payment_options_available` (avec `meta.reason_codes` triés : `no_merchant_address`, `merchant_address_provisioning`, `below_rail_minimum`, `rail_unavailable`, `scanner_unavailable`, `scanner_capacity_exhausted`, `matching_capacity_exhausted` — `merchant_address_provisioning` est transitoire et se résout généralement en une minute ou deux), `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Lire une facture

### `GET /v1/invoices/{id}`

Renvoie le résumé public de la facture, l’état de paiement visible par le payeur, l’identité de marque du projet et les instructions de paiement. **Aucune clé API requise** — les ids de facture sont des ids publics partageables et impossibles à deviner, utilisés dans les URL de liens de paiement ; c’est donc l’endpoint que les interfaces de paiement interrogent en continu (CORS autorise toute origine pour GET).

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

- `status` est le statut canonique de la facture, adossé aux événements de paiement et de règlement confirmés : `unpaid`, `partially_paid`, `paid`, `settling`, `settled` ou `review_required`.
- `review_required` signifie que la facture est en attente de vérification manuelle. Ce n’est **pas** un état payé — ne traitez pas la commande sur cette base, même si `amount_paid` semble suffisant.
- `checkout_status` est l’état dérivé côté payeur décrit plus haut ; les factures de production avec une preuve on-chain en attente affichent `confirming` tandis que le `status` canonique reste inchangé jusqu’à la confirmation du transfert.
- `transfers` est le journal de reçus côté payeur : les transferts entrants confirmés qui ont crédité cette facture, pour qu’un checkout puisse afficher chaque transaction on-chain et proposer un lien vers un explorateur de blocs. Chaque entrée porte `chain_namespace`, `chain_reference`, le `transaction_id` canonique, `event_index`, `amount` (en unités de la devise de la facture, à la même échelle à 18 décimales que `amount_paid` ; pour `direct_exact`, l’incrément de correspondance en est exclu) et `explorer_transaction_url` (ou `null`). Seuls les transferts confirmés apparaissent — un transfert en attente pourrait encore être annulé par un reorg — plafonnés aux 20 plus gros par montant, pour que des poussières envoyées à une adresse de dépôt publique ne puissent pas évincer le vrai paiement. Toujours présent : `[]` tant qu’aucun transfert n’est confirmé, et toujours `[]` pour les factures de test.
- Le `reference_id`, réservé à l’appelant, est omis ici ; seuls les champs de marque de `project` destinés au payeur sont renvoyés.
- Les ids mal formés comme les ids inconnus renvoient `404 invoice_not_found`.

## Simuler un paiement de test

### `POST /v1/invoices/{id}/test-payments`

Ajoute un paiement simulé à une facture de **test** et renvoie l’état de paiement mis à jour. Uniquement disponible avec les clés `sk_test_...` — c’est ainsi que vous déroulez la boucle complète unpaid → paid → webhook sans toucher à une chaîne.

```json
{ "amount": "5.0000", "reference_id": "test_payment_001" }
```

- `amount` est requis, doit être supérieur à zéro, avec au plus 15 chiffres entiers et 4 décimales (`5`, `5.0`, `5.0000` se normalisent en `5.0000`).
- `reference_id` est optionnel, 200 caractères max, et idempotent par facture : le réutiliser avec le même montant normalisé renvoie `200 OK` avec `meta.result: "reused"` ; un montant différent renvoie `409 test_payment_reference_conflict`.
- Les paiements partiels, complets et excédentaires sont autorisés : `partially_paid` tant que `0 < amount_paid < amount`, `paid` dès que `amount_paid >= amount`. La première transition vers `paid` déclenche un seul webhook logique `invoice.paid`, et chaque paiement créé incrémente `payment_revision`.
- La création est limitée à 300 par minute et 10 000 par jour et par projet.

Codes d’erreur : `401 invalid_secret_key`, `400 invalid_request`, `400 invalid_amount`, `404 invoice_not_found`, `409 project_archived`, `409 test_mode_required`, `409 test_payment_reference_conflict`, `413 request_body_too_large`, `429 rate_limited`, `500 server_misconfigured`.

## Webhooks

Configurez les URL de webhook dans le tableau de bord — test et production ont chacun leur propre URL et leur propre secret de signature.

### Événements

**`invoice.paid`** — envoyé quand une facture passe dans un état payé (`paid`, `settling` ou `settled`) :

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

**`invoice.payment_reversed`** — envoyé quand une facture précédemment payée repasse sous son montant (par exemple, un reorg de chaîne a retiré un transfert crédité). Même forme de payload, avec le `status` actuel de la facture, `amount_paid`, un `payment_revision` plus élevé et `fully_paid_at: null`. Traitez-le comme l’invalidation du signal de traitement antérieur, selon votre propre politique métier.

- `review_required` ne déclenche jamais `invoice.paid`. L’événement n’est créé qu’une fois la vérification résolue vers un état payé.
- Une vraie séquence payé → annulé → payé livre `invoice.paid`, `invoice.payment_reversed`, puis un nouvel `invoice.paid`, chacun avec son `payment_revision` résultant.
- `reference_id` et `fully_paid_at` sont nullables mais toujours présents ; `return_url` et les instructions de paiement sont volontairement absents. Réconciliez côté serveur par id de facture plus `reference_id`.

### Vérifier les signatures

Chaque webhook sortant inclut :

```http
Content-Type: application/json
Invoq-Signature: t=...,v1=...
```

`t` est un horodatage Unix en secondes. `v1` est le HMAC-SHA256 hexadécimal en minuscules de `<t>.<raw_body>`, avec pour clé le secret de webhook propre au mode. Vérifiez le **corps brut de la requête** avant de l’analyser — re-sérialiser le JSON peut en changer les octets et invalider la signature. Rejetez les horodatages hors de votre fenêtre de tolérance au rejeu. (Les SDK officiels fournissent un helper `verifyWebhook`.)

### Livraison et nouvelles tentatives

- Les POST de livraison expirent après 10 secondes.
- Chaque réponse non 2xx — **y compris les redirections et les 4xx** — ainsi que les erreurs réseau et les délais dépassés donnent lieu à de nouvelles tentatives avec un backoff borné : 1 minute, 5 minutes, 30 minutes, puis 2 heures, chaque délai avec jusqu’à 20 % de jitter, pour un maximum de cinq tentatives au total.
- La livraison est **au moins une fois** et peut arriver dans le désordre : dédupliquez par `id` d’événement, conservez l’instantané portant le plus grand `data.invoice.payment_revision`, et répondez `2xx` rapidement (faites le travail après avoir accusé réception).

## Mise en production

Une fois que la boucle du [Démarrage rapide](#démarrage-rapide) fonctionne avec votre webhook de test (un tunnel comme ngrok ou cloudflared convient pour le développement local) :

1. Créez une clé `sk_live_` dans le tableau de bord.
2. Configurez votre URL de webhook de production dans le tableau de bord.
3. Changez la clé dans la configuration de votre serveur. Rien d’autre ne change : mêmes endpoints, mêmes formes — les factures de production portent désormais de vraies `payment_options`.

Les factures de test et les paiements de test ne touchent jamais une chaîne et ne comptent jamais comme de vrais paiements.

## Support

- X : [@invoqmoney](https://x.com/invoqmoney) · 中文 [@invoqcn](https://x.com/invoqcn)
- Discord : https://discord.gg/V8cVrg4dET
- Telegram : https://t.me/invoqmoney
- E-mail : help@invoq.money
