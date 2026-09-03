# 35. JS/TS: `@indexnowkit/nuxt` (Nuxt 3.x/4.x, Nitro)

Nuxt module (`@nuxt/module-builder`, `defineNuxtModule`). Регистрация в `nuxt/modules` реестре.

## Что даёт

- `runtimeConfig.indexnow` = схема 02; `INDEXNOW_KEY` через `NUXT_INDEXNOW_KEY`.
- Nitro server route `/[key].txt` (добавляется через `addServerHandler`).
- Nitro plugin: `nitroApp.hooks.hook('request', ...)` открывает Collector-область на запрос,
  `afterResponse` делает flush; dispatcher по presets: Cloudflare (`event.waitUntil`),
  Vercel (`@vercel/functions` `waitUntil` при наличии), Node (`setImmediate`).
- Composable серверный `useIndexNow()` (`server/utils`, auto-import) → инстанс core.
- Nuxt Content: hook `content:file:afterParse` не про публикацию; отдельно опция
  `content: { submitOnBuild: 'diff' }` использует `sitemap --diff` при `nuxt generate`
  (кеш `.nuxt/indexnow-cache.json`).
- Интеграция с `@nuxtjs/sitemap`: если установлен, команда/endpoint `POST /api/_indexnow/sitemap`
  (защищён `runtimeConfig.indexnow.adminToken`) обходит его `sitemap.xml`.
- ORM: любой из 31–33 с `dispatcher` из nitro-контекста.

## Статус экосистемы

`@nuxtjs/seo` / Nuxt AI Ready документирует IndexNow как рецепт (ручной endpoint), не
автоотправку (nuxt-seo#306). Перепроверить при реализации; если появится нативно, наш
модуль ограничивается ORM-триггерами и `diff`-режимом.
