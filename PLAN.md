# better-webhook — Plan d'implémentation

> Webhooks sortants fiables. Enregistrement endpoints, delivery avec retry, signature HMAC, logs.

**Conventions** : [ECOSYSTEM.md](../better-agnostic/ECOSYSTEM.md) dans better-agnostic.

---

## Problème résolu

Les applications doivent notifier des systèmes externes (Stripe, Zapier, apps clientes) via des webhooks. Sans abstraction :

- **Retry manuel** : Si le serveur distant est down, il faut gérer la queue, le backoff, la dead-letter.
- **Signature** : Le client doit vérifier que le payload vient bien de nous. HMAC à implémenter.
- **Logs** : Qui a reçu quoi, quand, succès ou échec — pour le support et le debug.
- **Filtrage** : Un endpoint ne veut que les events `invoice.paid`, pas les autres.
- **Replay** : Rejouer un webhook qui a échoué après correction.

better-webhook unifie : `register(endpoint)`, `deliver(topic, payload)`. Retry exponentiel, signature HMAC sur chaque payload, logs de delivery, filtrage par topic, replay manuel.

---

## Concepts clés

| Concept | Description |
|---------|-------------|
| **WebhookEndpoint** | `{ id, url, secret, topics?, headers?, enabled }`. Un endpoint = une URL qui reçoit les payloads. |
| **Topic** | Type d'event : `invoice.paid`, `user.created`. Un endpoint peut s'abonner à des topics spécifiques. |
| **Delivery** | Une tentative d'envoi. Stockée avec status (pending, sent, failed), attempts, lastError. |
| **Signature** | Header `X-Webhook-Signature` : HMAC-SHA256 du payload avec le secret de l'endpoint. Le client vérifie. |

---

## Exemple d'usage

```ts
import { createWebhook } from "@better-webhook/core";
import { createDrizzleAdapter } from "@better-webhook/adapter-drizzle";
import { createRedisAdapter } from "@better-webhook/adapter-redis";
import { createSignaturePlugin } from "@better-webhook/plugin-signature";
import { createRetryPlugin } from "@better-webhook/plugin-retry";
import { createFilterPlugin } from "@better-webhook/plugin-filter";
import { createHonoHandler } from "@better-webhook/handler-hono";

const webhook = createWebhook({
  adapter: createDrizzleAdapter({ db }),
  queueAdapter: createRedisAdapter({ url: process.env.REDIS_URL }),
  plugins: [
    createSignaturePlugin({ algorithm: "sha256" }),
    createRetryPlugin({
      attempts: 5,
      backoff: "exponential",
      initialDelay: "30s",
      deadLetter: true,
    }),
    createFilterPlugin(),
  ],
});

// Enregistrer un endpoint (via API ou manuellement)
await webhook.register({
  url: "https://client.com/webhooks",
  secret: "whsec_xxx",
  topics: ["invoice.paid", "invoice.updated"],
  headers: { "X-Custom": "value" },
});

// Livrer — le système envoie à tous les endpoints abonnés au topic
await webhook.deliver("invoice.paid", {
  id: "inv-123",
  amount: 150,
  status: "paid",
  createdAt: "2024-01-15T10:00:00Z",
});

// Logs
const logs = await webhook.getDeliveryLogs({ topic: "invoice.paid", status: "failed" });

// Replay
await webhook.replay(deliveryId);

// Routes HTTP (handler)
app.route("/webhooks", createHonoHandler(webhook));
// POST /webhooks/endpoints — register
// GET  /webhooks/endpoints — list
// GET  /webhooks/deliveries — logs
// POST /webhooks/deliveries/:id/replay
```

---

## Fonctionnement détaillé des plugins

### plugin-signature (P0)

**Problème** : Le client doit vérifier que le payload vient bien de nous et n'a pas été altéré.

**Fonctionnement** :
- Chaque payload est signé avec HMAC-SHA256 (ou configurable) en utilisant le `secret` de l'endpoint.
- Header envoyé : `X-Webhook-Signature` ou `X-Webhook-Signature-256` (format `t=timestamp,s=signature` pour replay attack protection).
- Le client vérifie : `crypto.createHmac('sha256', secret).update(body).digest('hex')` et compare.
- Option `timestampTolerance` : rejeter les payloads trop vieux (replay attack).

### plugin-retry (P0)

**Problème** : Le serveur distant peut être temporairement down. Il faut réessayer avec backoff.

**Fonctionnement** :
- Queue (Redis, pg) pour les deliveries en attente.
- Backoff exponentiel : 30s, 1m, 2m, 4m, 8m (configurable).
- Après N tentatives échouées : dead-letter. Option `onExhausted` pour alerter.
- `webhook.retry.getQueue()`, `webhook.retry.getDeadLetters()`.
- Les retries sont asynchrones (fire-and-forget après la première tentative).

### plugin-filter (P1)

**Problème** : Un endpoint ne veut que certains topics. Ex: uniquement `invoice.paid`, pas `user.created`.

**Fonctionnement** :
- Chaque endpoint a un champ `topics?: string[]`. Si vide ou absent = tous les topics.
- Support de patterns : `invoice.*` pour tous les events invoice.
- À la delivery : on filtre les endpoints qui sont abonnés au topic avant d'envoyer.
- Évite d'envoyer du bruit aux clients qui n'en ont pas besoin.

### plugin-replay (P2)

**Problème** : Un webhook a échoué. Après correction (bug côté client), on veut le rejouer.

**Fonctionnement** :
- `webhook.replay(deliveryId)` : réinjecte le payload dans la queue avec le même endpoint.
- Compte comme une nouvelle tentative (retry count reset ou incrémenté selon la config).
- Utile pour le support : "on a corrigé le bug, pouvez-vous rejouer la delivery X ?".

### plugin-rate-limit (P2)

**Problème** : Ne pas spammer un endpoint qui répond lentement ou qui a des limites.

**Fonctionnement** :
- Limite par endpoint : max N deliveries par minute.
- Si dépassé : mise en queue, delivery différée.
- Adapter Redis pour le compteur.

### plugin-audit (P1)

**Problème** : Tracer les deliveries pour compliance et debug.

**Fonctionnement** :
- Chaque delivery (success, failed, retry) → `audit.log('webhook.delivered', ...)`.
- Bridge better-audit.
- Dead-letter → alerte (optionnel).

---

## Format du payload et headers

```
POST {endpoint.url}
Content-Type: application/json
X-Webhook-Signature: t=1705312800,s=abc123...
X-Webhook-Topic: invoice.paid
X-Webhook-Delivery-Id: del_01J8X...
X-Webhook-Timestamp: 1705312800

{ "id": "inv-123", "amount": 150, ... }
```

Le client vérifie :
1. Le timestamp est récent (éviter replay).
2. La signature correspond au body + secret.

---

## Adapters (providers)

| Adapter | Provider | Usage |
|---------|----------|-------|
| `adapter-memory` | — | Dev, tests |
| `adapter-drizzle` | PostgreSQL, SQLite | Endpoints + delivery logs |
| `adapter-redis` | Redis | Queue de retry (ou pg avec pg-boss/similar) |

---

## Structure des packages (kebab-case)

```
packages/
  core/
  types/
  errors/
  adapter-memory/
  adapter-drizzle/
  adapter-redis/
  handler-hono/
  handler-express/
  handler-fastify/
  handler-next/
  plugin-signature/
  plugin-retry/
  plugin-filter/
  plugin-replay/
  plugin-rate-limit/
  plugin-audit/
  client/
  cli/
```

---

## TODO

- [ ] Scaffold monorepo
- [ ] Core : types, adapter, register, deliver
- [ ] adapter-memory, adapter-drizzle
- [ ] adapter-redis (queue)
- [ ] plugin-signature, plugin-retry
- [ ] plugin-filter, plugin-audit
- [ ] Handlers (Hono, Express, Fastify, Next)
- [ ] plugin-replay, plugin-rate-limit
- [ ] client, cli
- [ ] Conformance, E2E, docs
