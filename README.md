# better-webhook

A headless, **pure TypeScript** webhook engine. Reliable outgoing webhooks with retry, HMAC signature, and adapters — without coupling to a specific framework or database. Same philosophy as better-auth — but for webhook.

## Install

```bash
bun add @better-webhook/core @better-webhook/adapter-memory
```

## Usage

```ts
import { createWebhook } from "@better-webhook/core";
import { createMemoryAdapter } from "@better-webhook/adapter-memory";

const webhook = createWebhook({
  adapter: createMemoryAdapter(),
  plugins: [
    createSignaturePlugin({ algorithm: "sha256" }),
    createRetryPlugin({ attempts: 5, backoff: "exponential" }),
  ],
});

await webhook.register({ url: "https://client.com/webhooks", secret: "whsec_xxx", topics: ["invoice.paid"] });
await webhook.deliver("invoice.paid", { id: "inv-123", amount: 150 });
```

## Packages

| Package | Description |
|---------|-------------|
| `@better-webhook/core` | Engine, register, deliver, adapter interface |
| `@better-webhook/types` | WebhookEndpoint, DeliveryLog |
| `@better-webhook/errors` | WebhookError, DeliveryError |
| `@better-webhook/adapter-memory` | In-memory adapter for dev/test |
| `@better-webhook/adapter-drizzle` | PostgreSQL, SQLite via Drizzle |
| `@better-webhook/adapter-redis` | Retry queue |
| `@better-webhook/handler-hono` | Hono bridge |
| `@better-webhook/handler-express` | Express bridge |
| `@better-webhook/handler-fastify` | Fastify bridge |
| `@better-webhook/handler-next` | Next.js bridge |
| `@better-webhook/plugin-signature` | HMAC signature |
| `@better-webhook/plugin-retry` | Retry with backoff |
| `@better-webhook/client` | HTTP client |
| `@better-webhook/cli` | CLI |

## Development

```bash
bun install
bun run build
bun run test
bun run lint
bun run typecheck
```

## License

MIT — see [LICENSE](LICENSE)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

## Code of Conduct

See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)
