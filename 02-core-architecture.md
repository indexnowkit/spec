# 02. Архитектура core (языконезависимо)

Каждый языковой core реализует одни и те же компоненты с одними именами. Различаются
только идиомы (интерфейс vs протокол vs trait). Адаптеры используют только эти
контракты, никогда HTTP напрямую.

## Компоненты

```
 ┌──────────────┐  submit(urls)          ┌────────────┐   flush()   ┌──────────────┐
 │ ChangeSource │ ─────────────────────▶ │ Collector  │ ──────────▶ │  Dispatcher  │
 │ (ORM hook,   │                        │ (per-req   │             │ sync | queue │
 │  CLI, user)  │                        │  buffer)   │             └──────┬───────┘
 └──────┬───────┘                        └────────────┘                    │
        │ created/updated/deleted                  ▲                       ▼
 ┌──────▼────────────┐                             │              ┌──────────────────┐
 │ ObjectChangeHandler│ classify per rule          │              │ Submitter        │
 │  + ChangeClassifier│                            │              │ debounce→batch→  │
 └──────┬─────────────┘                            │              │ Client           │
        │ resolveRule(subject, rule, event)        │              └────────┬─────────┘
 ┌──────▼───────┐   reads   ┌──────────────┐       │                       ▼
 │ UrlResolver  │◀──────────│  UrlRule[]   │───────┘              ┌──────────────────┐
 │  (guarded)   │           │ (per class)  │                      │ Client (HTTP)    │
 └──────────────┘           └──────────────┘                      │ + KeyProvider    │
                                                                  │ + Throttle       │
                                                                  └──────────────────┘
```

`UrlRule` — единица объявления (одно семейство публичных URL класса), `ObjectChangeHandler` — единица
работы ORM-хука (одно правило × одно событие жизненного цикла).

### Client

Чистая обёртка над протоколом (01-protocol.md).

- `submit(host, urls[]) -> Result` для одного батча ≤ maxUrls, один engine.
- `submitAll(urls[]) -> Result[]`: группирует по host, режет по maxUrls, шлёт в каждый
  сконфигурированный engine.
- Вход: `KeyProvider`, `HttpTransport` (абстракция HTTP клиента языка), `ClientOptions`
  (engines, timeout, userAgent, method).
- `Result { engine, host, urls[], status: 'ok'|'pending'|'failed'|'skipped', httpCode?, reason?, error?,
  retryable: bool, retryAfter?, endpoint }`.
- `reason` — машиночитаемая причина любого неуспеха, стабильный идентификатор для метрик и алертов:
  `disabled`, `dry_run`, `debounced`, `no_key`, `invalid_url` (ничего не отправлено) и `invalid_request`,
  `invalid_key`, `unprocessable`, `rate_limited`, `server_error`, `transport`, `unexpected` (отправка не удалась).
  `error` — человеческая формулировка, не API.
- Не бросает исключений на HTTP-ошибки протокола; бросает только на программные ошибки
  (невалидный ключ в конфиге, пустой список).
- User-Agent: `indexnowkit-<lang>/<version> (+https://indexnowkit.dev)`.

### KeyProvider

- `keyFor(host) -> key|null`, `keyLocationFor(host) -> url|null`.
- Реализации: `StaticKeyProvider` (из конфига/env), `MapKeyProvider` (host → key).
- Валидация ключа при построении: regex `^[A-Za-z0-9-]{8,128}$`, иначе
  `ConfigurationError` немедленно при старте приложения, не при первой отправке.
- `KeyGenerator.generate(length=32)` из CSPRNG, только `[a-f0-9]`.

### UrlRule и UrlResolver

`UrlRule` — скомпилированная, языконезависимая модель одного семейства публичных URL класса
(см. «Объявление модели» ниже). Класс объявляет **список** правил; всё остальное работает по правилам.

- `UrlResolver.resolve(subject, event) -> Url[]`, где `event ∈ {created, updated, deleted}`.
  Возвращает 0..n URL: одна сущность может иметь несколько страниц (локали, форматы, связанные объекты)
  или ни одной (черновик).
- Дефолтная реализация обходит правила класса и резолвит каждое через его источник. Пользовательский
  резолвер (`resolver:` в правиле или замена целиком) остаётся возможным.
- `explain(subject, event) -> ResolvedUrl[]`, где `ResolvedUrl { url, rule, class, event, locale? }`, —
  та же работа с сохранением происхождения: для `explain`-команд, логов и панелей профайлера.
- Контракт для `deleted`: адаптер обязан вызвать резолв до фактического удаления (pre-remove) и
  сохранить URL до post-commit.
- Guard-обёртка (`GuardedUrlResolver`) — единственная точка «объект → URL» для фасада и ORM-хуков:
  никогда не бросает, ошибку декларации или резолвера пишет в лог и возвращает пустой список.

### ObjectChangeHandler

Общий для всех адаптеров блок «объект изменился → URL»: поиск правил, классификация события
**по каждому правилу** и guard-резолв. Никогда не бросает.

- Хуки, срабатывающие до записи (нет идентификаторов): `createdEvents/updatedEvents/deletedEvents ->
  RuleEvent[]`, позже `resolve(subject, ruleEvent) -> ResolvedUrl[]`.
- Хуки, срабатывающие после записи (observer-ы, save-хуки): `created/updated/deleted -> ResolvedUrl[]`.
- Удаления и правила, переставшие применяться, резолвятся, пока живо старое состояние.

### Collector

- Буфер на единицу работы (HTTP-запрос, CLI-команда, job). Интерфейс: `add(urls)`, `all()`, `count()`,
  `isEmpty()`, `drain() -> UrlSet`, `reset()`. Заменяем: durable outbox, буфер на тенанта.
- Дедуп внутри буфера. Ключ дедупа: нормализованный URL.
- `reset()` очищает буфер **без доставки** (долгоживущие рантаймы) и обязан логировать предупреждение,
  если буфер был не пуст: единица работы закончилась без `flush()`, URL потеряны.
- Привязка к запросу: PHP request-scoped сервис, Python contextvar, JS AsyncLocalStorage,
  Go context, Java request scope/ThreadLocal, .NET scoped service.
- `flush()` вызывается адаптером в конце единицы работы (kernel.terminate,
  request_finished, `after()`, defer) **и только если транзакция закоммичена**.
  Точки commit-safety описаны в адаптерах.

### Dispatcher

- `dispatch(UrlSet)`. Два режима:
  - `sync`: вызывает Submitter немедленно (после ответа клиенту, где возможно). Ошибки
    только в лог. Ретраев нет.
  - `queue`: кладёт `SubmitUrlsMessage {urls[], attempt}` в очередь фреймворка; воркер
    вызывает Submitter; ретраи по политике 01-protocol.md.
- Дефолт: `sync`, если у фреймворка нет очереди «из коробки»; `queue`, если есть и она
  сконфигурирована (Messenger, Laravel Queue, Celery — см. адаптеры). Выбор явный в конфиге,
  автодетект только для дефолта.

### Submitter

Оркестратор: `submit(UrlSet) -> Result[]`.

1. Debounce: отбросить URL, отправленные менее `debounce.perUrl` секунд назад.
   `DebounceStore { lastSubmittedAt(url), markSubmitted(urls, at) }`. Реализации:
   `MemoryDebounceStore` (дефолт для CLI и тестов), адаптеры дают cache-backed
   (PSR-6/16, Django cache, Redis, Rails.cache).
2. Group by host, split by maxUrls.
3. Throttle: token bucket `throttle.maxRequestsPerMinute` (только в queue-режиме; в sync
   лимит достигается редко, но проверка есть).
4. `Client.submitAll`.
5. Mark submitted только для `ok|pending`.
6. Emit events/hooks: `onSubmitted(Result)`, `onFailed(Result)` для метрик пользователя.

### Logger / Metrics

- Core принимает абстрактный logger языка (PSR-3, `logging`, pino-совместимый объект,
  slog, SLF4J, ILogger).
- Уровни: success `debug`, pending(202) `info`, dry-run `info`, `enabled: false` `info`, 4xx `error` (403,
  400) / `warning` (422), 429/5xx `warning`, пятый подряд 403 на хост — один раз `critical`.
- Все сообщения начинаются с `indexnow: `; ключи маскируются везде, включая тела ответов и тексты
  исключений.
- Метрики строятся из `Result`: низкокардинальные метки `status`, `engine`, `reason`, `http_code`,
  `retryable`. `host` намеренно не входит (неограничен в мультитенантных установках).
- Причины «тихих» отказов (правило не подписано на событие, `when` ложен, `fields` не совпал) пишутся на
  уровне `debug`: это и есть ответ на вопрос «почему ничего не отправилось».

## Единая схема конфигурации

Имена ключей одинаковы во всех языках (snake_case в PHP/Python/Ruby/Go, camelCase в JS/Java/.NET).

```yaml
indexnow:
  enabled: true                  # false = ничего не отправлять, но всё логировать как dry-run
  key: '%env(INDEXNOW_KEY)%'     # дефолтный ключ
  hosts:                         # опционально, host → key либо host → {key, key_location, base_url}
    blog.example.com:
      key: '%env(INDEXNOW_KEY_BLOG)%'
      base_url: 'https://blog.example.com'   # генерация URL этого хоста вне HTTP-контекста
      engines: [yandex]                      # движки только для этого хоста
  strict_hosts: false            # true = хосты вне base_url/hosts пропускаются, а не шлются под дефолтным ключом
  previous_key: null             # ключ до ротации: файл ключа отдаётся, отправка под ним не идёт; также hosts.<host>.previous_key
  key_location: null             # опционально, полный URL файла ключа (на хосте base_url)
  base_url: 'https://www.example.com'   # для генерации абсолютных URL вне HTTP-контекста (CLI, воркеры)
  environment: '%env(APP_ENV)%'  # не-production без ключа включает dry_run вместо падения
  production_environments: [prod, production]  # какие имена считать production
  max_url_length: 2048           # длиннее — invalid_url
  engines: [api]                 # api | yandex | bing | naver | seznam | yep | custom URL | alias из engine_aliases
  engine_aliases: {}             # corp: https://index.corp.example/indexnow
  locale_hosts: {}               # en: www.example.com, de: example.de — правило с locales и без host генерирует локаль на её хосте
  dispatch: sync                 # sync | queue | none
  queue:                         # специфично для адаптера (transport name, queue name)
  batch:
    max_urls: 10000
  debounce:
    per_url: 600                 # секунд; 0 = выключить
    store: memory                # memory | cache | <service id>
    key_prefix: indexnowkit_     # префикс ключей общего кэша
  throttle:
    max_requests_per_minute: 60
  http:
    timeout: 10
    user_agent: null             # override
  serve_key_file: true           # адаптер регистрирует GET /{key}.txt
  dry_run: false                 # лог вместо HTTP
  logging:
    max_urls: 20                 # сколько URL перечислять в строке лога (0 = только счётчики)
    forbidden_escalation: 5      # подряд 403 на хост до уровня critical
    max_body: 300                # байт тела ответа движка в строке лога
    levels: {}                   # переопределение уровня по исходу: ok, pending, rate_limited, debounced, ...
  retry:                         # RetryPolicy для очередей и RetryingSubmitter
    max_attempts: 3
    base_delay: 60
    multiplier: 2.0
    max_delay: 3600
    server_error_delay: 5
  resolver:
    max_via_depth: 3             # глубина via:
    max_via_fanout: 100          # ширина одного via:
  collector:
    max_urls: 0                  # ранний flush при накоплении N URL (0 = только в конце)
    detect_leaks: true           # warning на shutdown о несброшенных URL
```

Всё это — `Config::OPTIONS`; адаптер отдаёт свой массив в `Config::fromArray()` и проверяет лишние ключи
`Config::unknownOptions()`. Адаптер-специфичные блоки (Symfony: `messenger`, `key_file`, `sitemap`, `doctrine`,
`flush`, `logging.channel`, `profiler`) он вырезает сам. Блок `queue` адаптер называет словарём своего транспорта
(Symfony: `messenger`, значение `dispatch: messenger` вместо `queue`); `retry.*` он может не выставлять, если повторы
делает очередь фреймворка.

Обязательные проверки при старте (framework check / boot validation):
- `key` или `hosts` задан, если `enabled`.
- Формат ключа.
- `base_url` абсолютный, если `dispatch: queue` (воркер не имеет request-контекста).
- `dry_run` включён автоматически, если env не production и ключ пустой (лог с
  предупреждением, не падение), чтобы dev-окружения не били в API.

## Публичный API для пользователя (единый)

| Действие | PHP | Python | JS | Ruby |
|---|---|---|---|---|
| Отправить URL вручную | `$indexNow->submit([$url])` | `indexnow.submit([url])` | `await indexNow.submit([url])` | `IndexNow.submit([url])` |
| Отправить сущность | `$indexNow->submitEntity($post)` | `indexnow.submit_object(post)` | `indexNow.submitRecord('Post', post)` | `IndexNow.submit_record(post)` |
| Сгенерировать ключ | `bin/console indexnow:key:generate` | `manage.py indexnow_key` | `npx indexnowkit key` | `rails indexnow:key` |
| Проверить ключ и конфиг | `indexnow:check` | `manage.py check` | `npx indexnowkit check` | `rails indexnow:check` |
| Отправить sitemap | `indexnow:submit-sitemap <url>` | `manage.py indexnow_sitemap` | `npx indexnowkit sitemap` | `rails indexnow:sitemap` |

`check` делает: валидирует конфиг, скачивает `https://<host>/<key>.txt`, сравнивает тело,
шлёт тестовый POST с `dry_run`. Это первая команда в README, она снимает 80% issue «не работает».

## Объявление модели: правила URL

Класс объявляет **список правил** (`UrlRule`), по одному на семейство публичных страниц. Синтаксис
объявления свой в каждом языке; скомпилированная модель одна и та же. Адаптер обязан привести своё
объявление к этому виду, дальше вся логика (классификация событий, guard-ы, локали, `via`, дедуп)
общая.

```
UrlRule {
  name:       string                    # стабильный id для логов, отладки и переопределения в наследниках
  source:     route | resolver | via | url | urls    # ровно один
  route:      string?                   # имя маршрута фреймворка
  params:     map<string, ParamValue>   # accessor | value | formatted | call
  resolver:   string?                   # id сервиса/класса UrlResolver
  via:        string?                   # accessor к связанному объекту или коллекции
  url:        string?                   # accessor, возвращающий string | iterable<string> | null
  urls:       string[]                  # литеральные URL
  when:       string[]                  # конъюнкция: все accessor-ы должны быть истинны
  whenFields: string[]                  # поля модели, от которых зависит when (для определения старого состояния)
  fields:     string[]                  # фильтр изменённых полей; [] = любые
  events:     (created|updated|deleted)[]
  locales:    'current' | 'all' | string[]
  host:       string | ParamValue | null
}
```

`ParamValue` — закрытый набор из четырёх источников: `accessor` (строка: свойство, геттер,
`is`/`has`-метод, точечный путь, `self`), `value` (константа), `formatted` (accessor + формат даты),
`call` (метод с аргументами; плейсхолдеры локали и хоста подставляются для каждого генерируемого
URL). Всё, что не выражается этими четырьмя, — повод написать `resolver`. Языки с замыканиями в
объявлении (JS, Ruby) могут принимать функцию вместо accessor-строки: это деталь фронтенда, `UrlRule`
от этого не меняется.

### Политика уровня класса

Класс может задать общие `when`, `fields`, `events`, `locales`. `when` **конъюнктивен**: правило
добавляет своё условие к условию класса, а не заменяет его (страница черновика не публична, что бы ни
говорило правило). Остальные три — значения по умолчанию: правило, задавшее своё, выигрывает
(`null` = наследовать, `[]` = без фильтра).

### Наследование

Правила собираются от корня иерархии к листу. Правило с тем же `name`, что и у предка, **заменяет**
его; правило с новым `name` добавляется. Политика класса сливается по полям, ближайшее объявление
выигрывает. Интерфейсы и трейты не сканируются.

### Имена полей

`fields` и `whenFields` — **имена полей модели** в том виде, в каком их пишет разработчик, никогда не
имена колонок БД. Doctrine `getEntityChangeSet()`, Django `update_fields`/FieldTracker и Prisma
`args.data` дают именно их. Объявленное поле совпадает с изменённым, если они равны или одно является
точечным префиксом другого, поэтому `fields: ['address']` ловит изменение встроенного объекта
`address.city`.

### Видимость и события

Видимость правила вычисляется до и после изменения. Переход `true → false` — `deleted` **для URL этого
правила** (остальные правила класса продолжают жить); `false → true` — `created`; без перехода —
`updated`, отфильтрованный по `fields`. Если старое значение `when` невычислимо, но изменилось хотя бы
одно поле из `when`/`whenFields`, считается, что видимость переключилась: ложное срабатывание стоит
одного лишнего запроса, пропуск оставляет мёртвую страницу в индексе. Поле, стоящее за accessor-ом,
ищется по имени, затем по конвенции (`isPublished` → `published` → `is_published`, `getStatus` →
`status`).

Удаление объекта, для которого правило неприменимо, **не отправляется**: страница и так не была
публичной.

| Событие ORM | `when` до | `when` после | `fields` совпал | Событие правила |
|---|---|---|---|---|
| insert | — | true | — | `created` |
| insert | — | false | — | нет |
| update | true | true | да | `updated` |
| update | true | true | нет | нет |
| update | true | false | не важно | **`deleted`** (резолв по текущему состоянию, до записи) |
| update | false | true | не важно | `created` |
| update | false | false | — | нет |
| delete | — | true | — | `deleted` (резолв до удаления) |
| delete | — | false | — | нет |

### `via`

`via` переотправляет страницы связанного объекта (комментарий → его пост, товар → его категория).
Цель всегда резолвится как `updated` — её страница существует независимо от того, что случилось с
источником. Глубина ограничена (по умолчанию 3), количество связанных объектов на правило — тоже
(по умолчанию 100).

### Пример соответствия синтаксисов

| | PHP | Python | JS |
|---|---|---|---|
| правило | повторяемый `#[IndexNow(...)]` | повторяемый декоратор `@indexnow(...)` | элемент `rules: [...]` |
| политика | `#[IndexNowDefaults(...)]` | `@indexnow_defaults(...)` | поля модели вне `rules` |
| accessor | `'category.slug'` | `"category.slug"` | `'category.slug'` или `p => p.category.slug` |
| константа | `new Value('html')` | `Value("html")` | `{ value: 'html' }` |
| из метода | `#[IndexNowUrl]` на методе | `get_absolute_url()` через миксин | `url: p => ...` |

### Правила, регистрируемые в рантайме

Модели, которые не могут нести атрибуты (типы записей CMS, чужие классы), регистрируют правила в коде.
Реестр реализует тот же контракт чтения правил поверх внутреннего ридера: статически по классу либо
фабрикой, решающей по объекту. Зарегистрированные правила заменяют то, что вернул бы внутренний ридер,
наследники их наследуют.

## Поведение при ошибках

- Никогда не пробрасывать исключение в пользовательский код из ORM-хука. Всё в лог.
- Исключение допустимо только из явного `submit()` вызванного пользователем, и только
  для программных ошибок (нет ключа, невалидный URL). HTTP-ошибки возвращаются как `Result`.
- `403` больше N раз подряд (N=5) → лог `critical` один раз, дальше `warning`, отправка
  продолжается (ключ мог быть только что развёрнут, 202 → 200 приходит с задержкой).

## Версионирование

- Core semver независимо от адаптеров. Адаптер объявляет `^major` core.
- Единый CHANGELOG-формат (Keep a Changelog), релиз через tag `<package>@<version>`.
