# 02. Архитектура core (языконезависимо)

Каждый языковой core реализует одни и те же компоненты с одними именами. Различаются
только идиомы (интерфейс vs протокол vs trait). Адаптеры используют только эти
контракты, никогда HTTP напрямую.

## Компоненты

```
 ┌──────────────┐  submit(urls, reason)  ┌────────────┐   flush()   ┌──────────────┐
 │ ChangeSource │ ─────────────────────▶ │ Collector  │ ──────────▶ │  Dispatcher  │
 │ (ORM hook,   │                        │ (per-req   │             │ sync | queue │
 │  CLI, user)  │                        │  buffer)   │             └──────┬───────┘
 └──────────────┘                        └────────────┘                    │
        │ resolve(entity) ▲                                                ▼
 ┌──────┴───────┐         │                                     ┌──────────────────┐
 │ UrlResolver  │─────────┘                                     │ Submitter        │
 │ (per model)  │                                               │ debounce→batch→  │
 └──────────────┘                                               │ throttle→Client  │
                                                                └────────┬─────────┘
                                                                         ▼
                                                                ┌──────────────────┐
                                                                │ Client (HTTP)    │
                                                                │ + KeyProvider    │
                                                                └──────────────────┘
```

### Client

Чистая обёртка над протоколом (01-protocol.md).

- `submit(host, urls[]) -> Result` для одного батча ≤ maxUrls, один engine.
- `submitAll(urls[]) -> Result[]`: группирует по host, режет по maxUrls, шлёт в каждый
  сконфигурированный engine.
- Вход: `KeyProvider`, `HttpTransport` (абстракция HTTP клиента языка), `ClientOptions`
  (engines, timeout, userAgent, method).
- `Result { engine, host, urlCount, status: 'ok'|'pending'|'failed', httpCode, error?, retryable: bool }`.
- Не бросает исключений на HTTP-ошибки протокола; бросает только на программные ошибки
  (невалидный ключ в конфиге, пустой список).
- User-Agent: `indexnowkit-<lang>/<version> (+https://indexnowkit.dev)`.

### KeyProvider

- `keyFor(host) -> key|null`, `keyLocationFor(host) -> url|null`.
- Реализации: `StaticKeyProvider` (из конфига/env), `MapKeyProvider` (host → key).
- Валидация ключа при построении: regex `^[A-Za-z0-9-]{8,128}$`, иначе
  `ConfigurationError` немедленно при старте приложения, не при первой отправке.
- `KeyGenerator.generate(length=32)` из CSPRNG, только `[a-f0-9]`.

### UrlResolver

- `resolve(subject, event) -> Url[]` где `event ∈ {created, updated, deleted}`.
  Возвращает 0..n URL: одна сущность может иметь несколько страниц (локали, форматы) или
  ни одной (черновик).
- Реализации в адаптерах: из атрибута/декоратора модели, из колбэка пользователя, из
  `getAbsoluteUrl()`-подобного метода.
- Контракт для `deleted`: адаптер обязан вызвать `resolve` до фактического удаления
  (pre-remove) и сохранить URL до post-commit.
- Фильтр публикации: `shouldSubmit(subject, event) -> bool` (например `status == published`).
  Дефолт: true. Переход published → draft трактуется как `deleted` (URL тот же, страница
  теперь 404); адаптер не обязан это определять сам, но даёт хук.

### Collector

- Буфер на единицу работы (HTTP-запрос, CLI-команда, job). `add(url, reason)`, `drain() -> UrlSet`.
- Дедуп внутри буфера. Ключ дедупа: нормализованный URL.
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
- Уровни: success `debug`, pending(202) `info`, 4xx `error` (403) / `warning` (422),
  429/5xx `warning` (с retry) / `error` (после исчерпания).
- Опциональный `MetricsSink` с двумя счётчиками: `indexnow_submitted_urls_total{engine,status}`,
  `indexnow_requests_total{engine,http_code}`.

## Единая схема конфигурации

Имена ключей одинаковы во всех языках (snake_case в PHP/Python/Ruby/Go, camelCase в JS/Java/.NET).

```yaml
indexnow:
  enabled: true                  # false = ничего не отправлять, но всё логировать как dry-run
  key: '%env(INDEXNOW_KEY)%'     # дефолтный ключ
  hosts:                         # опционально, host → key (мультисайт)
    blog.example.com: '%env(INDEXNOW_KEY_BLOG)%'
  key_location: null             # опционально, полный URL файла ключа
  base_url: 'https://www.example.com'   # для генерации абсолютных URL вне HTTP-контекста (CLI, воркеры)
  engines: [api]                 # api | yandex | bing | naver | seznam | yep | custom URL
  dispatch: sync                 # sync | queue | none
  queue:                         # специфично для адаптера (transport name, queue name)
  batch:
    max_urls: 10000
  debounce:
    per_url: 600                 # секунд; 0 = выключить
    store: memory                # memory | cache | <service id>
  throttle:
    max_requests_per_minute: 60
  http:
    timeout: 10
    user_agent: null             # override
  serve_key_file: true           # адаптер регистрирует GET /{key}.txt
  dry_run: false                 # лог вместо HTTP
```

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

## Объявление модели (единый смысл, разный синтаксис)

Минимальная форма: указать, как из объекта получить URL.

- PHP: `#[IndexNow(route: 'post_show', params: ['slug' => 'slug'])]` или `#[IndexNow(resolver: PostUrlResolver::class)]`.
- Python: `class Post(IndexNowModel, models.Model)` c `get_absolute_url()` или `indexnow.register(Post, url=lambda p: ...)`.
- JS: `indexNow.model('Post', { url: p => `/posts/${p.slug}`, when: p => p.published })`.
- Ruby: `index_now url: ->(p) { post_url(p) }, if: :published?`.

Общие опции: `when`/`if` (фильтр публикации), `events` (подмножество created/updated/deleted),
`on_fields` (отправлять только если изменились указанные поля; дефолт: любые).

## Поведение при ошибках

- Никогда не пробрасывать исключение в пользовательский код из ORM-хука. Всё в лог.
- Исключение допустимо только из явного `submit()` вызванного пользователем, и только
  для программных ошибок (нет ключа, невалидный URL). HTTP-ошибки возвращаются как `Result`.
- `403` больше N раз подряд (N=5) → лог `critical` один раз, дальше `warning`, отправка
  продолжается (ключ мог быть только что развёрнут, 202 → 200 приходит с задержкой).

## Версионирование

- Core semver независимо от адаптеров. Адаптер объявляет `^major` core.
- Единый CHANGELOG-формат (Keep a Changelog), релиз через tag `<package>@<version>`.
