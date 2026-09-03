# 12. PHP: `indexnowkit/symfony-bundle`

Symfony 6.4 LTS, 7.x. Тип `symfony-bundle`. Зависимости: `indexnowkit/core`, `symfony/config`,
`symfony/console`, `symfony/dependency-injection`, `symfony/event-dispatcher`, `symfony/framework-bundle`,
`symfony/http-foundation`, `symfony/http-kernel`, `symfony/routing`. В `suggest`: `indexnowkit/doctrine` +
`doctrine/doctrine-bundle` (хуки сущностей), `symfony/messenger`, `symfony/http-client`, `nyholm/psr7`,
`symfony/web-profiler-bundle`. Flex-рецепт в `recipe/`.

## Установка

```bash
composer require indexnowkit/symfony-bundle
composer require indexnowkit/doctrine     # автоотправка при изменении сущностей
```

```yaml
# config/packages/indexnowkit.yaml (из рецепта)
indexnowkit:
    key: '%env(INDEXNOW_KEY)%'
    base_url: '%env(INDEXNOW_BASE_URL)%'
when@dev:  { indexnowkit: { dry_run: true } }
when@test: { indexnowkit: { dry_run: true } }
```

Алиас расширения — `indexnowkit`. `AbstractBundle`: `configure(DefinitionConfigurator)` по схеме 02 плюс
Symfony-специфичные блоки, `loadExtension()`, `prependExtension()`.

## Дерево конфигурации

`enabled`, `key`, `base_url`, `key_location`, `hosts` (`host => key` либо `{key, key_location, base_url}`),
`strict_hosts`, `engines`, `dispatch` (`auto|sync|messenger|none`), `messenger.{bus,transport}`,
`batch.max_urls`, `debounce.{per_url,store}`, `throttle.max_requests_per_minute`,
`http.{timeout,user_agent,client}`, `key_file.{enabled,path,host,cache_max_age}`, `serve_key_file`
(устаревший алиас `key_file.enabled`), `dry_run`, `doctrine.{enabled,listener_priority,connections}`.

Проверки на этапе компиляции: `dispatch: messenger` без `base_url`; `hosts` без `base_url`; `strict_hosts`
без единого известного хоста; пустой `engines`; литеральный `key` не по формату; литеральный `base_url` не
абсолютный; неизвестный движок; `key_file.path` без `{key}`; `dispatch: messenger` без symfony/messenger.
Значения вида `%env(...)%` пропускаются (резолвятся в рантайме).

Ошибка в env-значении **не** ломает контейнер и не бросает из flush: `ConfigFactory` ловит
`ConfigurationException`, пишет `critical` и возвращает `Config(enabled: false, dryRun: true)`; точную ошибку
печатает `indexnow:check`.

`hosts` — array-нода, её нельзя заполнить из одной env-переменной; плейсхолдеры ставятся на каждый ключ.

## Объявление сущности

```php
#[ORM\Entity]
#[IndexNowDefaults(when: 'isPublished', fields: ['slug','title','body','published'])]
#[IndexNow(route: 'post_show', params: ['slug' => 'slug'])]
#[IndexNow(route: 'post_amp', params: ['slug' => 'slug'], when: 'hasAmp')]
#[IndexNow(via: 'category')]
class Post { ... }
```

`SymfonyRouteUrlResolver` реализует `RouteUrlResolverInterface`:

- `locales('all')` → `%kernel.enabled_locales%`, `'current'` → `[null]`, список → как задан;
  локаль подставляется в маршрут параметром `_locale`.
- `generate()` → `UrlGeneratorInterface::ABSOLUTE_URL`. Вне HTTP-запроса контекст берётся из `base_url`;
  правило с `host:` генерируется на `hosts.<host>.base_url` (иначе `https://<host>`). Контекст роутера
  временно подменяется и восстанавливается в `finally`. Ошибка роутера превращается в
  `ConfigurationException` с именем маршрута.

`#[IndexNow(resolver: ...)]` резолвится через `ContainerResolverLocator`: любой сервис
`UrlResolverInterface` автоконфигурируется тегом `indexnowkit.url_resolver` и доступен по своему id
(при стандартной автоконфигурации `App\` это FQCN). Класс без зависимостей инстанцируется на месте; класс с
зависимостями, не зарегистрированный под этим id, даёт понятную ошибку.

## Проводка Doctrine

- `IndexNowListener` — сервис с тегами `doctrine.event_listener` на `onFlush` и `postFlush`,
  `priority: %indexnowkit.doctrine.listener_priority%` (по умолчанию `-100`, после Gedmo).
- `IndexNowMiddleware` — тег `doctrine.middleware`.
- `doctrine.connections` ограничивает оба тега перечисленными соединениями (пусто = все).
- Хуки включаются только при наличии `indexnowkit/doctrine` **и** `doctrine/doctrine-bundle`, при
  `doctrine.enabled: true` и `enabled: true`. Иначе `indexnow:check` печатает предупреждение, а не молчит.
  Параметр `indexnowkit.doctrine_hooked` отражает фактическое состояние.

## Commit → Collector → Dispatcher

- `Collector` — сервис с тегом `kernel.reset`; `reset()` на непустом буфере логирует `warning`.
- `FlushListener` слушает `kernel.terminate` (priority `-1000`, до `ProfilerListener`), `console.terminate`
  и `Messenger\Event\WorkerMessageHandledEvent`. Фасад достаётся из service locator, поэтому запрос, ничего
  не собравший, его не инстанцирует.
- `dispatch: sync` → `SyncDispatcher`; `messenger` → `MessengerDispatcher` (`DispatchAfterCurrentBusStamp`);
  `none` → `NullDispatcher`. `auto` → `messenger`, если установлен symfony/messenger и объявлены
  `framework.messenger.transports`, иначе `sync`; при `enabled: false` — всегда `none`.
- `SubmitUrlsMessage(list<string> $urls)` + `#[AsMessageHandler] SubmitUrlsHandler`. Ретраибельный исход
  (429/5xx/сеть) → `RecoverableMessageHandlingException`; на Symfony ≥ 7.2 в неё передаётся задержка из
  `Retry-After` (мс). 400/403/422 финальны и только логируются. Мёртвые письма — штатный
  `framework.messenger.failure_transport`, своей команды пакет не даёт.
- `messenger.transport: async` заставляет `prependExtension()` добавить
  `framework.messenger.routing[SubmitUrlsMessage] = async`; параметр `indexnowkit.messenger_routed`
  показывает, действительно ли сообщение маршрутизировано.

## HTTP-клиент

`TransportFactory`: сервис `http.client` — PSR-18 используется как есть, `symfony/http-client` (включая
scoped-клиенты) оборачивается в `Psr18Client`, `null` — автообнаружение с `http.timeout` и без редиректов.
Сам транспорт — `LazyTransport`: клиент строится при первом запросе, поэтому запрос без отправок ничего не
стоит, а `indexnow:check` сообщает об отсутствии клиента как об ошибке проверки.

## Key file

Роут `indexnowkit_key_file`: `GET %indexnowkit.key_file.path%` (по умолчанию `/{key}.txt`), requirement
`[A-Za-z0-9-]{8,128}`, host из `%indexnowkit.key_file.host%`. `KeyFileController` спрашивает
`KeyFileResponder::bodyForKey($key, $request->getHost())`, поэтому отдаётся только ключ **этого** хоста;
иначе 404. Заголовки — `text/plain; charset=utf-8` и `Cache-Control: public, max-age=key_file.cache_max_age`
(300 с по умолчанию, чтобы ротация ключа расходилась быстро). Выключается `key_file.enabled: false`.

## Команды

`indexnow:key:generate [-l|--length] [--alphanumeric] [--write-env[=FILE]] [--force]`,
`indexnow:check [--live] [--host]`,
`indexnow:submit <urls...> [-f|--force] [--dry-run] [--json]`,
`indexnow:submit-entity <class> [ids...] [--event] [--limit] [--explain] [-f|--force] [--dry-run] [--json]`,
`indexnow:explain <class> <id> [--event]`,
`indexnow:sitemap [sitemap] [--changed-since] [--allow-foreign-hosts] [-f|--force] [--dry-run] [--json]`.

`indexnow:sitemap` читает потоком (`SitemapReader` складывает документы во временные файлы) и отправляет каждые
`batch.max_urls` URL, сводя результаты в `ResultSummary` (строка на engine/host/status со счётчиками `urls` и
`batches`): список URL целиком в памяти не живёт. Блок конфига `sitemap` — `enabled` (false = команды и сервиса
`indexnowkit.sitemap_reader` нет), `url` (дефолт аргумента вместо `<base_url>/sitemap.xml`), `max_depth`,
`max_sitemaps`, `max_bytes`, `allow_foreign_hosts`, `spool` (auto|disk|memory: временный файл либо память на
read-only FS), `spool_dir`, `fetch_retries`. При обрыве посреди sitemap недобранная порция отправляется до выхода
с кодом 1. `indexnow:check` печатает, куда идёт spool, и почему temp-каталог непригоден. Команда зависит от
`SitemapSourceInterface` (alias на `indexnowkit.sitemap_reader`): приложение декорирует или подменяет источник;
аргумент может быть локальным путём / `file://`. Полная карта точек расширения — `docs/extending.md` бандла.

Symfony-only ноды поверх общей схемы 02: `messenger.{bus, transport, delay, stamps}`, `key_file.*` (включая
`route_name`; маршрут строит сервис `indexnowkit.key_file_routes`), `sitemap.*`, `doctrine.*`, `logging.channel`,
`profiler.enabled`, `flush.{priority, console_priority}`. Alias'ы для декорирования: `ClientInterface`
(`indexnowkit.client`), `Command\EntityLoaderInterface`, `Command\SubmitterFactoryInterface`,
`Command\ResultFormatterInterface`, `Check\CheckerInterface` (+ автоконфигурация `Check\CheckInterface` тегом
`indexnowkit.check`), остальные — в `docs/configuration.md`. Файл ключа отдаётся с `Vary: Host` при непустом
`hosts`. `tests/Functional/CoreConformanceTest.php` гоняет `CoreConformanceTestCase` ядра против собранного фасада.

`--force` и `--dry-run` собирают отдельный `Submitter` через `SubmitterFactory` (`NullDebounceStore` и/или
`Config::with(dryRun: true)`), не трогая сервис приложения. Вывод — таблица со столбцом `reason` либо JSON.
`indexnow:submit-entity` и `indexnow:explain` регистрируются только при доступной Doctrine.

`indexnow:explain` проходит весь путь решения одной сущности: правила → подписка на событие → `when` →
`fields` → URL → нормализация → host/ключ/файл ключа → дебаунс. Ничего не отправляет.

## Профайлер

`IndexNowDataCollector` (late collector) + `ResultRecorder`, слушатель `Submitter`: панель показывает
собранные URL, реальные результаты (включая синхронную отправку на `kernel.terminate`, которая происходит
после клонирования коллекторов), режим dispatch и факт маршрутизации в Messenger, движки, `base_url`,
`strict_hosts`, окно дебаунса и URL файла ключа по каждому хосту (ключ замаскирован).

## Логирование

Канал Monolog `indexnow` (`IndexNowKitLoader::LOG_CHANNEL`) на всех логирующих сервисах: клиент, submitter,
throttle, collector, резолверы, change handler, фасад, dispatcher, messenger handler, listener Doctrine.

## Тесты

`symfony/framework-bundle` test kernel, ORM sqlite, Messenger `in-memory://`, транспорт подменяется на
`IndexNowKit\Testing\FakeTransport` алиасом `indexnowkit.transport` в compiler pass (тот же рецепт
документирован для пользователей). Матрица 6.4/7.x, PHP 8.2–8.4. Сценарии H01–H03, H06, A01/A02/A04, C13.

## Flex recipe

`recipe/`: `config/packages/indexnowkit.yaml` (ключ, base_url, закомментированные примеры мультидомена,
Messenger и scoped-клиента, `dry_run: true` в dev и test) и `config/routes/indexnowkit.yaml`.
PR в `symfony/recipes-contrib` — открытая задача (91).
