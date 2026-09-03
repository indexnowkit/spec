# 30. JS/TS: `@indexnowkit/core`

Runtime-agnostic: Node ≥ 20 (рекомендация 22), Bun, Deno, Cloudflare Workers, Vercel Edge.
Зависимости: **ноль**. Глобальный `fetch`. ESM + CJS через tsup (`exports` map, `types`),
`sideEffects: false`. Монорепо `indexnowkit/js`: pnpm workspaces + changesets, npm trusted
publishing (OIDC), vitest projects, tsc strict, biome.

## Публичный API

```ts
import { createIndexNow, Engine } from '@indexnowkit/core';
const indexNow = createIndexNow({ key: process.env.INDEXNOW_KEY!, baseUrl: 'https://www.example.com', engines: [Engine.Api] });
const results = await indexNow.submit(['/posts/a', 'https://www.example.com/b']);
indexNow.model('Post', { url: (p: Post) => `/posts/${p.slug}`, when: p => p.published, fields: ['slug','title','body','published'] });
await indexNow.submitRecord('Post', post, 'updated');
```

## Модули

| Файл | Контракт из 02 |
|---|---|
| `client.ts` | `Client`, `Transport = (req: {url, body, headers, timeoutMs}) => Promise<{status, body}>`, `fetchTransport` |
| `key.ts` | `KeyProvider`, `staticKeyProvider`, `mapKeyProvider`, `generateKey()` через `crypto.getRandomValues` (Web Crypto, есть везде) |
| `url.ts` | `normalizeUrl`, `hostOf`, `UrlResolver<T> = (record: T, event: ChangeEvent) => string | string[] | null | Promise<...>` |
| `registry.ts` | `ModelRegistry`: `model(name, { url, when, events, fields })`, `resolve(name, record, event)` |
| `collector.ts` | `Collector` на `AsyncLocalStorage` (Node/Bun/Deno/CF с `nodejs_als` флагом); fallback: явный `collector = indexNow.collector()` объект, передаваемый адаптером |
| `debounce.ts` | `DebounceStore { lastSubmittedAt(url), markSubmitted(urls, at) }`, `memoryDebounceStore(ttl)`, `kvDebounceStore` для CF KV/Redis-подобного `{get,set}` интерфейса |
| `submitter.ts` | debounce → group host → chunk → throttle → client |
| `dispatcher.ts` | `Dispatcher = (urls: string[]) => void | Promise<void>`; `syncDispatcher`, `deferredDispatcher(schedule: (fn) => void)` для `after()`/`waitUntil`; `queueDispatcher(enqueue)` |
| `sitemap.ts` | `sitemapUrls(url, { transport })` c минимальным XML-парсером (regex-free tokenizer, sitemap index, gzip через `DecompressionStream`) |
| `logger.ts` | `Logger` интерфейс `{debug,info,warn,error}` (console по умолчанию, pino-совместимый) |

## Типизация

Один generic `UrlResolver<T>`. Адаптеры используют реальные типы своих ORM (Prisma `Prisma.PostGetPayload`,
TypeORM entity classes) через generic-параметры, core их не знает.

## Collector без ALS

Cloudflare Workers требует `compatibility_flags = ["nodejs_als"]`; если ALS недоступен,
`indexNow.collect(async (collector) => {...})` явная область. Адаптеры фреймворков создают
область сами (Next: в `after`-обёртке; Nuxt: nitro request hook).

## CLI `@indexnowkit/cli`

`npx @indexnowkit/cli key|check|submit <urls>|sitemap <url> [--diff <cache.json>]`.
`sitemap --diff` хранит `{url: lastmod}` и шлёт только изменившиеся: основа для SSG и
GitHub Action `indexnowkit/action`.

## Conformance

`packages/core/test/conformance.test.ts` читает YAML (`yaml` в devDeps), mock-сервер из
`indexnowkit/spec` через `testcontainers` или `INDEXNOW_MOCK_URL`.
