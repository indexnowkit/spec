# 39. JS/TS: Strapi, Directus, Sanity

## `@indexnowkit/strapi` (Strapi 5)

- Document Service middleware в `register()`: `strapi.documents.use(async (ctx, next) => { const r = await next(); if (['publish','unpublish','create','update','delete'].includes(ctx.action)) ...; return r; })`.
  Только `publish`/`unpublish` по умолчанию (draft&publish); `create/update/delete` для
  контент-типов без draft&publish. Lifecycle hooks не используем (двойное срабатывание в v5).
- URL: `config.plugin('indexnow').urls[uid] = (doc, locale) => ...`; дефолт из Preview
  `getPreviewPathname`, если сконфигурирован.
- Commit: Document Service middleware выполняется после операции; транзакции Strapi (`strapi.db.transaction`)
  оборачивают отдельные операции; middleware после `next()` = после commit операции
  (проверить). Dispatch: `setImmediate`, опционально cron `strapi.cron` для батч-flush.
- Роут `GET /:key.txt` через plugin `server/routes` с `config: { auth: false }`, префикс
  `/api` убирается опцией `prefix: ''`.
- Scaffold: `@strapi/sdk-plugin`. Strapi Market submission.
- Существующий `strapi-indexnow` (0.0.4, 52/мес) минимален.

## `@indexnowkit/directus`

Hook extension (`defineHook`): `action('items.create'|'items.update'|'items.delete', ...)`,
`urlTemplate` per collection в env (`INDEXNOW_URLS='{"posts":"/posts/{slug}"}'`). Ключ-роут
через endpoint extension `/{key}.txt` (Directus endpoints под `/`? нет: под `/<id>`; для
корня нужен reverse-proxy правило, документируем). Плюс экспорт Flow JSON как no-code
альтернатива. Регистрация в Directus Extensions Registry (`directus-extension` keyword).

## Sanity

Не пакет, рецепт: GROQ webhook (`sanity-operation` header, только published) → наш
webhook-handler из 34/35 (`urlsFromPayload`). Опционально Sanity Function (проверить API).
