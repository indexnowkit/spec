# 12. PHP: `indexnowkit/symfony-bundle`

Symfony 6.4 LTS, 7.x. Тип `symfony-bundle`. Зависимости: `core`, `doctrine` (optional через
`suggest`, включается при наличии `doctrine/doctrine-bundle`), `symfony/http-kernel`,
`symfony/routing`, `symfony/console`. Flex-рецепт в `symfony/recipes-contrib`.

## Установка

```bash
composer require indexnowkit/symfony-bundle     # рецепт: config/packages/indexnow.yaml, .env INDEXNOW_KEY=, routes
```

```yaml
# config/packages/indexnow.yaml (из рецепта)
indexnow:
    key: '%env(INDEXNOW_KEY)%'
    base_url: '%env(INDEXNOW_BASE_URL)%'   # для CLI/воркеров; в HTTP берётся из Request
    dispatch: messenger                     # sync | messenger
    messenger: { bus: messenger.bus.default, transport: async }   # transport опционален (routing)
    debounce: { per_url: 600, store: cache.app }
    serve_key_file: true
```

`AbstractBundle` (6.1+): `configure(DefinitionConfigurator)` по схеме 02, `loadExtension()`,
`prependExtension()` добавляет routing `messenger` для `SubmitUrlsMessage`, если `dispatch: messenger`
и transport указан.

## Объявление сущности

```php
#[ORM\Entity]
#[IndexNow(route: 'post_show', params: ['slug' => 'slug'], when: 'isPublished', fields: ['slug','title','body','published'])]
class Post { ... }
```

`RouteUrlResolver` → `UrlGeneratorInterface::generate($route, $params, ABSOLUTE_URL)`.
Вне HTTP-контекста `RequestContext` берётся из `router.request_context` с `base_url`
(бандл выставляет `host`/`scheme` из `base_url` в `ConsoleCommandEvent` и в Messenger-воркере
через `WorkerMessageReceivedEvent`).

Локали: если маршрут локализован (`_locale` в requirements), резолвер генерирует URL для
всех `enabled_locales` (опция `locales: all|current|[list]` в атрибуте).

## Проводка Doctrine

- Listener из 11 как сервис с тегом `doctrine.event_listener` на `loadClassMetadata`,
  `onFlush`, `postFlush`, `priority: -100`, для всех `connections`/`entity_managers` из
  конфига `doctrine.orm` (или подмножество `indexnow.doctrine.entity_managers`).
- Middleware из 11 с тегом `doctrine.middleware` (DoctrineBundle ≥ 2.6).
- `ConnectionRegistry`: бандл инжектит `doctrine` ManagerRegistry, staging маппит по имени соединения.

## Commit → Collector → Dispatcher

- `Collector` request-scoped: сервис с `kernel.reset` (`ResetInterface`), вместе с
  `kernel.terminate` listener (после отправки ответа) и `console.terminate` listener.
  Messenger-воркер: `WorkerMessageHandledEvent` → flush.
- `dispatch: sync`: в terminate вызывается `Submitter` напрямую.
- `dispatch: messenger`: `SubmitUrlsMessage(array $urls, int $attempt = 0)` +
  `#[AsMessageHandler]`; при 429/5xx handler бросает `RecoverableMessageHandlingException`
  с `retryDelay`, retry_strategy transport'а. При 403/422/400 `UnrecoverableMessageHandlingException`.
  Сообщение диспатчится с `DispatchAfterCurrentBusStamp`, если мы внутри обработки другого
  сообщения (сохраняет гарантию `doctrine_transaction` middleware).

## Key file

Роут `indexnow_key_file`: `GET /{key}.txt`, requirements `[A-Za-z0-9-]{8,128}`, контроллер
сравнивает с `KeyProvider`, 404 иначе. Регистрируется через `RoutingConfigurator` в
`prependExtension` (routes loader `indexnow.routes`), выключается `serve_key_file: false`.
Приоритет роута ниже пользовательских (регистрируется последним).

## Команды

`indexnow:key:generate [--write-env]`, `indexnow:check`, `indexnow:submit <url>...`,
`indexnow:submit-entity <class> <id>`, `indexnow:sitemap <url|--presta>` (если установлен
`presta/sitemap-bundle`, обходит его dumper через `SitemapPopulateEvent`).

## Профайлер

`DataCollector` в Web Profiler: сколько URL собрано в запросе, что отправлено, результаты.
Панель ускоряет отладку «почему не отправилось».

## Тесты

`symfony/framework-bundle` test kernel, ORM sqlite, Messenger `in-memory://`. Матрица 6.4/7.2/7.3+.

## Flex recipe

`manifest.json`: `bundles`, `copy-from-recipe` (`config/packages/indexnow.yaml`), `env`
(`INDEXNOW_KEY=`, `INDEXNOW_BASE_URL=`). `.yaml`, 4 пробела в JSON, PR в `symfony/recipes-contrib`.
