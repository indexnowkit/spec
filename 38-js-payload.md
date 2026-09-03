# 38. JS/TS: `@indexnowkit/payload` (Payload CMS 3.x)

Plugin `(config) => config`.

```ts
plugins: [indexNowPlugin({ key: process.env.INDEXNOW_KEY, baseUrl, collections: { posts: { url: doc => `/posts/${doc.slug}` } } })]
```

- URL по умолчанию: `admin.livePreview.url` коллекции, если задан (та же сигнатура
  `(data, collectionConfig, locale) => url`); иначе `url` из опций плагина. Локали:
  `localization.locales` → URL для каждой, если `url` принимает locale.
- Хуки: `afterChange({ doc, previousDoc, operation })`, `afterDelete({ doc })`. Drafts:
  отправлять только `doc._status === 'published'`; переход `published → draft` = Deleted;
  `previousDoc` есть, `fields`-фильтр по diff.
- Commit-safety: Payload 3 оборачивает операцию в транзакцию (`req.transactionID`), хуки
  `afterChange` выполняются внутри неё; после commit есть `afterOperation`. Реализация:
  staging по `req.transactionID` в `afterChange/afterDelete`, flush в `afterOperation`
  (выполняется после commit транзакции операции). Подтвердить по исходникам при реализации
  (в исследовании помечено как недокументированное).
- Endpoint `{ path: '/:key.txt', method: 'get' }` в `config.endpoints` (Payload API идёт под
  `/api`; для корня нужен Next route `app/[key]/route.ts` из 34, плагин печатает подсказку).
- Dispatch: `after()` из next при наличии (Payload 3 = Next), иначе `setImmediate`.
- Конкурент `payload-cms-seo-plugin`: IndexNow выключен по умолчанию, побочная функция.
