# 34. JS/TS: `@indexnowkit/next` (Next.js 15+, App Router)

## Что даёт

1. Key file: `export { GET } from '@indexnowkit/next/key-route'` в `app/[key]/route.ts`
   (проверяет `params.key` против `INDEXNOW_KEY`, `text/plain`, 404 иначе). Альтернатива
   для Pages Router: `pages/api/[key].ts` re-export.
2. Dispatcher на `after()` из `next/server` (стабилен с 15.1): `nextDispatcher()` =
   `deferredDispatcher(fn => after(fn))`. На Vercel `after` делегирует `waitUntil`;
   self-hosted Node работает (Next сам держит процесс); Cloudflare через OpenNext
   поддерживает `after` (проверить версию `@opennextjs/cloudflare`).
3. Хелперы триггеров: `revalidateAndNotify(path | {tag}, urls)` вызывает `revalidatePath/Tag`
   и `after(() => indexNow.submit(urls))` одним вызовом; для Server Actions и Route Handlers.
4. Webhook route для headless CMS: `createIndexNowWebhook({ secret, urlsFromPayload })` →
   `POST /api/indexnow/webhook`, проверка HMAC/секрета, `revalidateTag`, submit.
5. ORM-триггер: сочетать с `@indexnowkit/prisma`/`drizzle` и `nextDispatcher`.

```ts
// lib/indexnow.ts
export const indexNow = createIndexNow({ key: process.env.INDEXNOW_KEY!, baseUrl: process.env.NEXT_PUBLIC_SITE_URL!, dispatcher: nextDispatcher() });
```

## Ограничения

- Edge runtime: core работает (fetch, Web Crypto), ALS доступен в Next edge.
- `next build` статическая генерация: отправку при билде не делаем (используйте
  `@indexnowkit/cli sitemap --diff` в CI после деплоя).
- Middleware/Proxy: не используем для dispatch.

## Конкуренты

`nextjs-indexnow` (198/мес), `next-indexnow` (59/мес): ручной submit без `after()`, без ORM.
