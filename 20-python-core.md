# 20. Python: core `indexnowkit` (волна P, шаг 1)

Статус: **переписана 2026-09-10** после выпуска PHP-семейства целиком (core 0.13.1 … cli 0.1.0, спека 17 §16, 19b); прежняя
редакция 2026-09-03 (4.3 КБ, до PHP-опыта) заменена. Порядок и гейт волны — спека 26; решения §9 там же (здесь — только
решения по самому core, §9). Ни строчки кода до «действуй».

## 0. Цель и границы

Один дистрибутив `indexnowkit` на PyPI, модуль `indexnowkit`: протокол (спека 01), модель правил и классификация
изменений (спека 02), конфигурация «одна схема, одно правило окружения», `check` с теми же кодами, что у PHP
(`core/docs/check-codes.md`), ключ-файл для WSGI и ASGI, sitemap, pre-flight (`verify`), история (`history`), файл
состояния, CLI без фреймворка и тест-кит conformance — **всё в одном дистрибутиве**, потому что в Python ничего из этого
не тянет зависимостей: HTTP — `urllib.request`, XML — `xml.etree`, robots — `urllib.robotparser`, HTML — `html.parser`,
состояние — `sqlite3`, CLI — `argparse`, тест-кит — `pytest` только в extra. PHP резал семейство на пять пакетов
(console/testing/sitemap/verify/history) из-за Composer-зависимостей и размера (спека 17 §1); в Python этих причин нет
(решение §9.1 спеки 26).

Адаптеры (21–25) используют только контракты этого модуля, никогда HTTP напрямую; их синтаксис объявления компилируется
в тот же `UrlRule`. Границы: без Google, без генерации sitemap, без дашборда (спека 00); без Docker-образа Python и без
zipapp (образ PHP уже покрывает «любой сайт», спека 19b; `uvx indexnowkit` — путь без установки, §3.12).

## 1. Факты (проверены 2026-09-10; PyPI JSON — `https://pypi.org/pypi/<name>/json`)

### 1.1. Версии Python и фреймворков

| Что | Версия | Дата | Python | Источник |
|---|---|---|---|---|
| Python 3.10 | security | EOL **2026-10** | — | devguide.python.org/versions |
| Python 3.11 | security | EOL 2027-10 | — | там же |
| Python 3.12 | security | EOL 2028-10 | — | там же (3.12 перешёл в security-фазу) |
| Python 3.13 | bugfix | EOL 2029-10 | — | там же |
| Python 3.14 | bugfix, 2025-10-07 | EOL 2030-10 | — | там же |
| Python 3.15 | prerelease, релиз 2026-10-01 | — | — | там же |
| Django 5.2.17 LTS | 2026-08-04 | extended support до 2028-04 | 3.10–3.14 (3.14 с 5.2.8) | djangoproject.com/download, releases/5.2 |
| Django 6.0.8 | 2026-08-04 | до 2027-04 | 3.12–3.14 | releases/6.0 (вышел 2025-12-03) |
| Django 6.1.1 | 2026-09-02 | до 2027-12 | 3.12–3.14 | releases/6.1 (вышел 2026-08-05) |
| Django 4.2 / 5.1 | EOL 2026-04 / 2025-12 | — | — | download: в таблице поддерживаемых их нет |
| Django 6.2 LTS | план 2027-04 | до 2030-04 | — | download |
| SQLAlchemy 2.0.52 | 2026-08-11 | — | ≥3.7 | PyPI |
| SQLAlchemy 2.1.0rc2 | 2026-09-08 (rc1 2026-08-31) | — | **≥3.11**; `greenlet` только в extra `[asyncio]` | PyPI, docs.sqlalchemy.org/en/21/changelog/migration_21 |
| FastAPI 0.141.1 | 2026-07-29 | — | ≥3.10; `starlette>=0.46`, `pydantic>=2.9` | PyPI |
| Starlette 1.6.0 | 2026-08-08 | — | ≥3.10 | PyPI (1.x — мажор; сайт starlette.io не резолвился 2026-09-10, release notes не прочитаны — §7.9) |
| Flask 3.1.3 | 2026-02-19 | — | ≥3.9 | PyPI |
| Flask-SQLAlchemy 3.1.1 | **2023-09-11** | — | ≥3.8 | PyPI (три года без релиза) |
| httpx 0.28.1 | 2024-12-06 | — | ≥3.8 | PyPI; `1.0.dev6` 2026-08-31 — 1.0 в пути |
| Wagtail 8.0 | 2026-08-25 | — | ≥3.10, `Django>=5.2` | PyPI |
| pytest 9.1.1 / pytest-django 4.14.0 / pytest-asyncio 1.4.0 | 2026-06-19 / 2026-08-10 / 2026-05-26 | — | ≥3.10; pytest-django: `django>=5.2`, классификаторы 5.2 и 6.0 | PyPI |
| mypy 2.3.1 / ruff 0.16.6 / uv 0.12.12 / hatchling 1.32.0 | 2026-08-15 / 2026-09-03 / 2026-09-09 / 2026-08-11 | — | — | PyPI |
| django-stubs 6.1.0 | 2026-08-12 | — | ≥3.11 | PyPI |
| Celery 5.6.3 / RQ 2.12.0 / Dramatiq 2.2.1 | 2026-03-26 / 2026-08-30 / 2026-09-02 | — | ≥3.9 / 3.10 / 3.10 | PyPI |
| django-tasks 0.12.0 (бэкпорт `django.tasks`, Django ≥4.2) / django-tasks-db 0.13.0 / django-tasks-rq 0.12.0 | 2026-02-06 / 2026-08-28 / 2026-02-06 | — | ≥3.10 | PyPI; djangoproject.com/community/ecosystem «Tasks» называет их и huey |

Следствия: 3.10 уходит из поддержки через месяц; Django 6.x требует 3.12; SQLAlchemy 2.1 — 3.11. PEP 604 (`X | Y`) —
3.10, PEP 695 (`class Foo[T]`, `type X = …`) — 3.12, PEP 696 (дефолты параметров типов) — 3.13 (peps.python.org). `enum.StrEnum`,
`typing.Self`, `tomllib`, `ExceptionGroup` — 3.11; `typing.override`, `itertools.batched` — 3.12.

### 1.2. Стандартная библиотека, на которую опирается core

- `urllib.request`: TLS проверяется по умолчанию (`ssl.create_default_context()`), `timeout` — аргумент `urlopen`, HTTP-ошибки —
  `urllib.error.HTTPError` (это и ответ: `.code`, `.headers`, `.read()`); редиректы отключаются подклассом `HTTPRedirectHandler`,
  чьи `http_error_30x` бросают `HTTPError`, через `build_opener` (docs.python.org/3/library/urllib.request). В 3.13 удалены
  `cafile`/`capath`/`cadefault` — только `context`.
- `contextvars.ContextVar`: изоляция на поток и на задачу asyncio, `set()` → `Token`, `reset(token)`; с 3.14 `Token` — контекстный
  менеджер; `copy_context().run()` для переноса в поток (docs.python.org/3/library/contextvars). Это `Collector` спеки 02
  («Python contextvar»).
- `asyncio`: event loop держит только слабые ссылки на задачи — `create_task()` без сохранения ссылки может быть собран GC до
  завершения; `asyncio.to_thread()` для блокирующего вызова (docs.python.org/3/library/asyncio-task).
- `xml.etree.ElementTree.iterparse` — потоковый разбор; уязвимости XML (billion laughs, quadratic blowup, large tokens) закрыты
  Expat ≥ 2.7.2, проверка — `pyexpat.EXPAT_VERSION` (docs.python.org/3/library/xml «XML vulnerabilities»). `defusedxml` не
  нужен: sitemap-документы без DTD, а лимит `sitemap.max_bytes` (как в PHP) режет бомбы декомпрессии.
- `urllib.robotparser.RobotFileParser` — robots.txt со стандартной семантикой `can_fetch(agent, url)`; `html.parser.HTMLParser` —
  `<meta name="robots">` и `<link rel="canonical">` для pre-flight; `secrets.token_hex(16)` — 32 hex-символа CSPRNG для ключа;
  `sqlite3` с `PRAGMA journal_mode=WAL` — файл состояния; `logging` — логгер `indexnowkit`; `argparse` — CLI; `dataclasses`
  (`frozen=True, slots=True`, `dataclasses.replace`) — `Config`, `Result`, `UrlRule`; `typing.Protocol` — интерфейсы.

### 1.3. Что PHP-опыт добавил в схему 02 и что в Python должно быть с 0.1.0

Из `Config::OPTIONS` (`php/packages/core/src/Config.php:111-121`, 46 ключей) и `core/docs/configuration.md`: `strict_hosts`,
`previous_key` (и `hosts.<host>.previous_key`), `key_location` и `hosts.<host>.{key, key_location, base_url, engines, previous_key}`,
`production_environments` + страховка «не production без ключа = `dry_run`» с флагом `dryRunExplicit` (`check` различает «не задан»
и «false явно»), `engine_aliases`, `locale_hosts`, `max_url_length`, `logging.{max_urls, forbidden_escalation, levels, max_body}`,
`retry.*` (5 ключей), `resolver.{max_via_depth, max_via_fanout}`, `collector.{max_urls, detect_leaks}`,
`normalizer.{strip_tracking_params, tracking_params, trailing_slash, sort_query}`, `key_file.{enabled, cache_max_age}`,
`debounce.{per_url, store, key_prefix}`, `http.{timeout, user_agent, client}`. Плюс `Config::arrayFromEnv()` — «только заданное»
для слияния «окружение поверх файла» и правило `INDEXNOW_<BLOCK>_<KEY>` для блоков (спека 19b §3.3, `cli/README.md`
«Configuration»). Булевы разбираются, не кастуются (`filter_var`-семантика: `false/0/no/off`) — урок W1 аудита 0.13.

Из поведения: `Reason` (16 значений: 5 skip ядра, 5 skip verify, 6 failed), `Result.metric_labels()` без host, `ForbiddenCounter`
в общем кэше (403 ×5 → одна `critical`), `check` как healthcheck с кодами и `--json` по `console/docs/check.schema.json`, `--strict`,
`--live`, `--sample`, `KeyFileResponder` (200/404, `text/plain; charset=utf-8`, `Cache-Control: public, max-age=300`, `Vary: Host` при
карте хостов или `strict_hosts`), `renamed()` — старые URL переименованной страницы как `deleted`, `via` с лимитами,
`ObjectChangeHandler` never-throws, `Collector.reset()` с warning о потерянных URL, `SubmissionStoreInterface` (S01–S08),
`SeenStoreInterface` и `sitemap --new-only`, `RetryPolicy`/`RetryingSubmitter`/`WorkerOutcome`.

### 1.4. Конкуренты на PyPI (снимок 2026-09-10, pypistats.org «recent»)

| Пакет | Версия, дата | Загрузок/мес | Что |
|---|---|---|---|
| `index-now-for-python` (jakob-bagterp, 8★) | 1.0.25, 2026-08-22, ≥3.11 | 2 400 | один URL / список / sitemap на один endpoint; без дебаунса, ORM, очередей |
| `wagtail-indexnow` (RealOrangeOne, 1★) | 0.2.0, 2025-08-06 | 1 111 | хуки `before/after_publish_page`, дебаунс по `last_published_at` (10 мин), ключ из `SECRET_KEY` через pbkdf2, `requests`; без unpublish/delete/batch/429 (спека 25) |
| `indexnow` (ajitjasrotia) | 0.9.9, 2023-08-05 | 271 | голый клиент; имя занято |
| `django-indexnow` (hckjck, GitHub 0★, 2026-03) | не на PyPI | — | сигналы + дедуп 60 с + middleware ключ-файла; stdlib-only; ближайший по идее, без commit-safety и батчей |

Имена `indexnowkit`, `indexnowkit-django`, `-sqlalchemy`, `-fastapi`, `-flask`, `-wagtail`, `-testing`, `-cli` — 404 на PyPI
(2026-09-10). Классификаторы существуют: `Framework :: Django :: 6.1`, `Framework :: Wagtail :: 8`, `Framework :: FastAPI`,
`Framework :: Flask`, `Framework :: AsyncIO`, `Programming Language :: Python :: 3.15` (pypi.org/classifiers).

### 1.5. Packaging и публикация

- **uv workspaces** (docs.astral.sh/uv/concepts/projects/workspaces): `[tool.uv.workspace] members` в корневом `pyproject.toml`,
  зависимости между членами `[tool.uv.sources] x = { workspace = true }` (editable), один `uv.lock`, `uv run --package`,
  `uv build --package`; ограничение — `requires-python` членов пересекается (все члены ≥ общего минимума).
- **hatchling** (hatch.pypa.io/latest/config/build): `[build-system] requires = ["hatchling"]`, `[tool.hatch.build.targets.wheel]
  packages = ["src/indexnowkit"]`, `py.typed` внутри пакета попадает в wheel как обычный файл.
- **uv build/publish** (docs.astral.sh/uv/guides/package): `uv build --no-sources` перед публикацией (проверка, что пакет собирается
  без `tool.uv.sources`), `uv publish` умеет trusted publishing без токена.
- **PyPI trusted publishing** (docs.pypi.org/trusted-publishers): «pending publisher» регистрируется **до** первого аплоада (имя
  проекта, владелец, репозиторий, имя файла workflow, environment); workflow — `permissions: id-token: write` +
  `pypa/gh-action-pypi-publish` (docs.github.com «Publishing to PyPI»: сборка `python -m build` в одной джобе, публикация из
  артефакта в другой, с `environment`). Несколько проектов могут указывать один и тот же файл workflow.

## 2. Принципы

1. **Одна схема, одно правило окружения, с первой версии.** `Config` — тот же словарь ключей, что `Config::OPTIONS` PHP (snake_case,
   вложенные блоки), `from_mapping()` / `from_env()` / `mapping_from_env()`, `INDEXNOW_<BLOCK>_<KEY>` для всего. Ничего «потом».
2. **Идиомы Python, не PHP.** `Protocol` вместо интерфейсов, frozen `dataclass` + `replace()` вместо `with()`, `StrEnum`,
   контекстные менеджеры (`with indexnow.collecting():`), `logging` с именованными логгерами, `contextvars`, type hints PEP 604;
   имена методов snake_case (`submit_objects`, `key_for`). Термины — словарь ядра (`core/docs/adapters.md` «Names»): `locale`,
   `subject`, `rule`, `event`.
3. **Никогда не бросать из хука.** `ObjectChangeHandler`, `GuardedUrlResolver`, диспетчеры, дебаунс-стор — логируют и продолжают
   (`core/docs/adapters.md` §15). Исключения — только из явных вызовов пользователя и только программные.
4. **Ноль зависимостей у дистрибутива.** Extras: `[httpx]` (async-транспорт), `[testing]` (pytest-кит). Ни `requests`, ни
   `pydantic`, ни `click`: `argparse` достаточно, а адаптеры оборачивают его в свой CLI.
5. **Машиночитаемо и честно, как в PHP.** Коды `check` — API, тексты — нет; `--json` по той же схеме; README с «Notes for AI
   assistants»; «Google: нет; IndexNow ≠ индексация».
6. **Sync первичен, async — та же логика.** `submit()` над `urllib`; `asubmit()` над `httpx.AsyncClient` (extra) или через
   `asyncio.to_thread(submit)` без него — одна `Submitter`-логика, транспорт — `Protocol` с sync и async вариантами.
7. **Тесты семейства — те же идентификаторы.** C01–C22, A01–A21 (+A05b/c, A10b), S01–S08, H01–H06 заморожены (спека 17 §7);
   тест-имена `test_c01_…`, проверка полноты как `ConformanceIdsTest`.

## 3. Дизайн

### 3.1. Дистрибутив и модули

`python/packages/indexnowkit/` (uv workspace member), `src/indexnowkit/`, `py.typed`, версия в `indexnowkit/__init__.py`
(`__version__`, hatch `[tool.hatch.version] path`). Модули (каждый ≤ 400 строк; PHP-аналог в скобках):

| Модуль | Содержимое |
|---|---|
| `indexnowkit` (`__init__`) | реэкспорт: `IndexNowKit`, `Config`, `Engine`, `Event`, `Reason`, `Result`, `ResultStatus`, `indexnow`, `indexnow_defaults`, `indexnow_url`, `registry`, `__version__` |
| `config` | `Config` (frozen dataclass, `OPTIONS: tuple[str, …]`, `from_mapping`, `from_env`, `mapping_from_env`, `to_mapping`, `unknown_options`, `replace`, `endpoints_for`, `base_url_for`, `base_host`, `key_file_headers`, `log_level`, `retry_policy`, `is_production`, `dry_run_explicit`); `config/parser.py`, `config/normalizer.py` — internal |
| `engine` | `Engine(StrEnum)` — 8 участников реестра (спека 01), `resolve_endpoint()` |
| `result` | `Result` (frozen), `ResultStatus`, `Reason` (16 значений, `is_skip`, `is_retryable`, `message`), `retryable_urls()`, `all_urls()`, `urls_where()`, `metric_labels()` |
| `client` | `Client` (POST всегда; GET — `http.method`), `ClientProtocol`, `ForbiddenCounter`, `FailureCache` |
| `http` | `Transport` protocol (`post(url, json, headers) -> Response`, `get(url) -> Response`, `download(url, sink)`), `AsyncTransport`, `Response` (`parse_retry_after`), `UrllibTransport` (без редиректов, лимиты тела 2 KiB POST / 50 MiB GET, `User-Agent`), `LazyTransport`, `TransportError`; `http/httpx.py` — `HttpxTransport` + `AsyncHttpxTransport` (импорт `httpx` внутри) |
| `key` | `KeyProvider` protocol (`key_for`, `key_location_for`, `is_known_key(key, host)`, `managed_hosts`), `StaticKeyProvider.from_config`, `KeyValidator` (`^[A-Za-z0-9-]{8,128}$`, `mask()`), `generate_key(length=32, hex=True)`, `KeyFileResponder` (`body_for_path`, `body_for_key`, `PATH_PATTERN`, `CONTENT_TYPE`, `DEFAULT_MAX_AGE=300`) |
| `wsgi`, `asgi` | `KeyFileApp` — WSGI-приложение и ASGI-приложение над `KeyFileResponder` (аналог PSR-15 `KeyFileRequestHandler`, спека 18 §3.5): как маршрут (`handle`) и как middleware (не ключ-файл → следующий app); `CollectorMiddleware` — scope коллектора на запрос и flush **после** отправки ответа (§3.7) |
| `url` | `UrlNormalizer` (RFC 3986 §6.2.2, punycode через `idna`-кодек stdlib, фрагмент, порт, dot-segments), `CanonicalUrlNormalizer` (`normalizer.*`), `InvalidUrlError`, `RouteOrigin` |
| `rules` | `UrlRule`, `RuleSet`, `RuleEvent`, `RuleSource`, `ParamValue` (`Accessor`, `Value`, `Formatted`, `Call`), `Condition`/`FieldCondition`/`Equals`, декораторы `@indexnow`, `@indexnow_defaults`, `@indexnow_url`, `RuleCompiler`, `RuleReader` (по `__dict__` класса и MRO — наследование по `name`), `RuleRegistry` (`register(cls, rules=…)`, фабрика по объекту), `ChangeClassifier` (таблица спеки 02), `ParamExtractor` + `SubjectReader` protocol (`supports`, `has`, `read`) — `plain()` для getattr/точечных путей |
| `resolve` | `UrlResolver` protocol, `RuleUrlResolver` (правила → URL: `route` через `RouteUrlResolver` protocol адаптера, `resolver`, `via` с `ViaWalk` и лимитами, `url`, `urls`, `locales`, `host`, `locale_hosts`), `GuardedUrlResolver` (never throws), `ObjectChangeHandler` (`created/updated/deleted`, `*_events`, `resolve`, `renamed`), `ResolvedUrl`, `CallableUrlResolver`, `ResolverLocator` |
| `collector` | `Collector` (contextvar-scope, дедуп, `max_urls`, `detect_leaks`), `collecting()` контекстный менеджер |
| `dispatch` | `Dispatcher` protocol, `SyncDispatcher`, `NullDispatcher`, `CallableDispatcher(fn)`, `ThreadDispatcher` (один воркер-поток, очередь, `atexit`-join с таймаутом, warning о потере при выходе), `AsyncTaskDispatcher` (задача на loop, строгая ссылка в `set`), `BatchingDispatcher` (чанки, `new_job_id`), `DispatcherFactory.from_config` |
| `debounce` | `DebounceStore` protocol (`last_submitted_at`, `mark_submitted`), `MemoryDebounceStore` (`threading.Lock`, bounded 50 000, TTL), `NullDebounceStore`, `SqliteDebounceStore` (файл состояния), `CacheDebounceStore` (над `CacheProtocol`: `get/set/add` — Django cache, любой объект с этими методами), `DebounceStoreFactory.from_config(config, locator)`, `is_shared()` |
| `throttle` | `Throttle` protocol, `TokenBucket` (`time.sleep`; `async_acquire` через `asyncio.sleep`), `NullThrottle`, `Clock` protocol + `SystemClock` |
| `submitter` | `Submitter` (дебаунс → группировка → чанки → throttle → client → mark → listeners → store), `SubmitterProtocol`, `Retry` (`RetryPolicy`, `RetryingSubmitter`, `WorkerOutcome`) |
| `submission` | `SubmissionStore` protocol (`record`, `recent`, `last_for`), `SubmissionRecord`, `NullSubmissionStore`, `ResultSummary`; `history/` — `SqliteSubmissionStore` (`purge`, `count`, `last`), `HistoryConfig` |
| `kit` | `IndexNowKit` фасад: `create(config, *, transport=None, …)` (только именованные), `submit(urls)`, `asubmit(urls)`, `submit_objects(objs, event=…)`, `collect(urls)`, `flush()`, `changes()`, `resolver()`, `explain(obj, event)`, атрибуты `config`, `keys`, `submitter`, `collector`, `dispatcher`, `transport` |
| `adapter` | `ConfigFactory` (`build`, `load`, `unknown_options`, `disabled`; параметры как в PHP: `owned_options`, `dispatch_modes`, `auto_dispatch`, `need_base_url`, `defaults`, `validate`, `check_command`, `ignore_blocks`), `Services` / `ServicesBuilder` (§3.9), `ObserverHelper` (`guard`, `deliver`, `remember_deletion`, `take_deletion`), `SubmitterFactory` (`--force`/`--dry-run`), `ConfigSource` protocol (`raw()`, `build()`, `packages()`) |
| `check` | `Checker`, `Check` protocol, `CheckItem` (`level`, `code`, `message`, `host`), `CheckReport`, `CheckLevel`, `StaticCheck`, `DebounceStoreCheck` (`PROBE_KEY`), `LocalesCheck`, `SampleGateCheck`, `SampleOptions`, `DispatchLine`, `ExpatCheck` (§7.5); коды — `core/docs/check-codes.md` PHP как эталон, файл `docs/check-codes.md` пакета |
| `console` | `Definitions` (аргументы/опции один раз, `argparse`-рендер; адаптеры (Django `add_arguments`, Flask click) конвертируют), `Vocabulary`, раннеры `CheckRunner`, `ConfigRunner`, `SubmitRunner`, `SubmitSubjectsRunner` + `SubjectLoader` protocol, `ExplainRunner`, `KeyGenerateRunner`, `KeyFileRunner`, `SitemapRunner`, `HistoryRunner`, `StatusRunner`, `ResultRenderer` (таблица/`--json`), `ExitCode` (0/1/2), `Io` (stdout/stderr, цвет по `isatty`) |
| `cli` | `python -m indexnowkit` и console-script `indexnowkit` (§3.12): `EnvConfigSource`, `.env` (свой парсер ~60 строк: `KEY=value`, кавычки, `#`; окружение процесса выигрывает), `--config file.json` или `file.toml` (`tomllib`), `State` (`.indexnow/state.sqlite`, `--state memory`), `Wiring` |
| `sitemap` | `SitemapReader` (`iterparse`, sitemap index рекурсивно с `max_depth`/`max_sitemaps`, `.gz` через `gzip`, текстовые списки, `lastmod`, `max_bytes`, spool на диск/память, `allow_foreign_hosts`, `fetch_retries`), `SitemapSource` protocol, `SitemapEntry`, `SeenStore` protocol + `SqliteSeenStore`, `SitemapConfig`, `SitemapSpoolCheck` |
| `verify` | `VerifyConfig`, `PageSignals` (`html.parser`: `noindex` meta/`X-Robots-Tag`, canonical, редиректы), `RobotsCache` (`urllib.robotparser`, TTL в кэше дебаунса), `VerifyingSubmitter` (декоратор: skip/replace/follow, `max_batch`, `time_budget`, 404/410 — как удаление), `SampleCheck` (`--sample`) |
| `testing` | без pytest: `FakeTransport`, `FrozenClock`, `RecordingDispatcher`, `MemoryCache`; `testing/conformance/` (импортирует `pytest` лениво): `CoreConformance`, `OrmConformance`, `SubmissionStoreConformance`, `KeyFileAssertions`, `CheckOutputAssertions`, `ReadmeAssertions`, `OptionalFeatureAssertions`; `testing/mock_server.py` — `MockIndexNowServer` (`http.server.ThreadingHTTPServer`, тот же контракт, что `router.php`: сценарии `X-Mock-Scenario`/`?scenario=`, `/_mock/requests`, `MOCK_KEYS`, `/large-document.xml[.gz]`) и pytest-фикстура `mock_indexnow` |

Логгер вместо `ArrayLogger` — pytest `caplog` (штатный); `ArrayLogger` не нужен (спека 26 §1, «иначе»).

### 3.2. Конфигурация

```python
@dataclass(frozen=True, slots=True)
class Config:
    enabled: bool = True
    key: str | None = None
    hosts: Mapping[str, str] = field(default_factory=dict)          # host -> key (нормализовано из строк и словарей)
    key_locations: Mapping[str, str] = ...; host_base_urls: ...; host_engines: ...; previous_keys: ...
    key_location: str | None = None
    base_url: str | None = None
    engines: tuple[str, ...] = (Engine.API,)
    dispatch: str = "sync"
    strict_hosts: bool = False
    environment: str | None = None
    production_environments: tuple[str, ...] = ("prod", "production")
    dry_run: bool = False
    dry_run_explicit: bool = True
    previous_key: str | None = None
    max_url_length: int = 2048
    batch_max_urls: int = 10_000
    debounce_per_url: int = 600; debounce_store: str | None = None; debounce_key_prefix: str = "indexnowkit_"
    throttle_max_requests_per_minute: int = 60
    http_timeout: float = 10.0; http_user_agent: str | None = None; http_client: str | None = None
    key_file_enabled: bool = True; key_file_cache_max_age: int = 300
    logging_max_urls: int = 20; logging_forbidden_escalation: int = 5; logging_levels: Mapping[str, str] = ...; logging_max_body: int = 300
    retry_max_attempts: int = 3; retry_base_delay: int = 60; retry_multiplier: float = 2.0; retry_max_delay: int = 3600; retry_server_error_delay: int = 5
    resolver_max_via_depth: int = 3; resolver_max_via_fanout: int = 100
    collector_max_urls: int = 0; collector_detect_leaks: bool = True
    engine_aliases: Mapping[str, str] = ...; locale_hosts: Mapping[str, str] = ...
    normalizer_strip_tracking_params: bool = True; normalizer_tracking_params: tuple[str, ...] = (); normalizer_trailing_slash: str = "keep"; normalizer_sort_query: bool = False

    OPTIONS: ClassVar[tuple[str, ...]]   # те же 45 dotted-ключей, что Config::OPTIONS PHP, без deprecated `serve_key_file`
```

- `__post_init__` — все проверки конструктора PHP (`Config.php:203-308`) с теми же текстами («факт — что можно — как починить»);
  `ConfigurationError`. `key` обязателен при `enabled and not dry_run and not hosts`.
- `Config.from_mapping(data: Mapping[str, Any]) -> Config` — вложенная форма схемы 02 (`{"debounce": {"per_url": 600}}`), строки
  коэрсятся (`"600"`, `"true"`), булевы **разбираются** (`true/1/yes/on` и `false/0/no/off`, пустая строка = не задано, иное —
  `ConfigurationError` с именем ключа); страховка `dry_run` вне production без ключа → `dry_run=True, dry_run_explicit=False`.
- `Config.from_env(env=None, prefix="INDEXNOW_")` = `from_mapping(mapping_from_env(env, prefix))`; `mapping_from_env()` отдаёт
  **только заданное**, строками, по правилу `INDEXNOW_<KEY_PATH>` для ядра **и** блоков (`INDEXNOW_DEBOUNCE_PER_URL`,
  `INDEXNOW_SITEMAP_MAX_DEPTH`, `INDEXNOW_HISTORY_STORE`, `INDEXNOW_VERIFY_ENABLED`), `INDEXNOW_HOSTS="host=key,host2=key2"`,
  `INDEXNOW_ENGINES="yandex,bing"`, `INDEXNOW_ENV` иначе `APP_ENV`/`DJANGO_ENV`? — нет: только `INDEXNOW_ENV`, иначе `APP_ENV`
  (как PHP); адаптер подставляет своё (`settings.DEBUG` → §21).
- `to_mapping()` — всё с разрешёнными значениями (для `config --json`, ключи маскируются раннером); `unknown_options(data, allowed)` —
  спуск во вложенные блоки (урок core 0.13.1, спека 19b п. 10).
- `replace(**changes)` — `dataclasses.replace` с валидацией; неизвестное имя → `ConfigurationError` со списком.
- Блоки пакетов: `SitemapConfig`, `VerifyConfig`, `HistoryConfig` — те же ключи, что в PHP (`sitemap.*` 9, `verify.*` 11,
  `history.*` — `store: null|sqlite|<адаптер>`, `limit`, `key_prefix`, `sqlite.path`, `table`, `retention_days`; `pdo.*` → `sqlite.*`
  и адаптерные `django.*` в 21), каждый со своим `OPTIONS`, `from_mapping`, `load_or_disabled(block, logger, check_command)`.

### 3.3. Протокол, клиент, результат

- `Engine(StrEnum)`: `API = "api"`, `YANDEX`, `BING`, `NAVER`, `SEZNAM`, `YEP`, `INTERNETARCHIVE`, `AMAZON` с endpoint'ами спеки 01;
  `Engine.resolve_endpoint(name_or_url)` принимает имя, алиас или `https://` URL (plain `http://` только на loopback — mock).
- `Client.submit(host, urls) -> Result`, `submit_all(urls) -> list[Result]`: группировка по host, чанки `batch_max_urls`, каждый
  endpoint из `endpoints_for(host)`; тело `{"host", "key", "keyLocation"?, "urlList"}`; коды → `Result` по таблице спеки 01;
  никаких исключений на HTTP-статус; `TransportError` → `failed/transport/retryable`; тело ответа — первые `logging_max_body` байт
  в лог; ключ маскируется везде (`KeyValidator.mask`: 4 символа + `…`).
- `Result` frozen: `engine`, `host`, `urls: tuple[str, ...]`, `status: ResultStatus`, `http_code`, `error`, `retryable`,
  `retry_after`, `endpoint`, `reason: Reason | None`; конструкторы `Result.ok/skipped/failed`; `NO_ENGINE = "none"`.
- `ForbiddenCounter` — в кэше дебаунса (`<prefix>403.<host>`, TTL 1 ч), пятый подряд 403 — одна `critical` на флот.

### 3.4. Модель правил

Синтаксис Python компилируется в тот же `UrlRule` (спека 02 «Объявление модели»):

```python
from indexnowkit import indexnow, indexnow_defaults, indexnow_url, Value

@indexnow_defaults(when="is_published", fields=["slug", "title", "published"])
@indexnow(route="post_detail", params={"slug": "slug"})                       # правило "post_detail" (name = route по умолчанию)
@indexnow(name="amp", route="post_amp", params={"slug": "slug", "format": Value("amp")}, when="has_amp")
@indexnow(name="home", urls=["/"])
class Post: ...

class Comment:
    @indexnow(via="post")                                                     # переотправить страницы поста
    ...

@indexnow_url                                                                 # метод, возвращающий str | Iterable[str] | None
def public_urls(self) -> list[str]: ...
```

- Декораторы **накладываемые** (несколько на класс), хранят список в `__indexnow_rules__` класса (собственный атрибут, не
  наследуемый — `RuleReader` идёт по MRO от корня к листу и сливает по `name`); `@indexnow_defaults` — политика класса
  (`when` конъюнктивен, остальные — умолчания). Accessor — строка (`"category.slug"`, `"is_published"`, `"self"`) **или**
  callable; callable не объясним по имени (`explain` печатает `<function …>`), строка — предпочтительна и единственно
  допустима в `when_fields`/`fields`.
- `ParamValue`: `Accessor`, `Value`, `Formatted(accessor, fmt)` (`strftime`), `Call(method, *args)` с плейсхолдерами `Placeholder.LOCALE`,
  `Placeholder.HOST`. Закрытый набор (Sealed в `bc.md`).
- `registry.register(cls, rules=[UrlRule(...)] | [indexnow(...)], defaults=…)` — рантайм-реестр для чужих классов; заменяет
  прочитанное с класса; наследники наследуют. `registry.register_factory(fn)` — по объекту (типы записей CMS).
- `ChangeClassifier` — таблица спеки 02 (insert/update/delete × `when` до/после × `fields`), поиск поля за геттером по конвенции
  (`is_published` → `published`; Python: `is_x`/`has_x`/`x` → `x`, `x` property → `_x`? — нет, только `is_/has_` префиксы и само имя).
  `when_fields` — явное указание.
- `ParamExtractor(*readers)` с `SubjectReader` protocol; `plain()` — `getattr` + точечный путь + вызов метода без аргументов;
  адаптеры добавляют ридеры (Django: `attname`/`_loaded_values`, SQLAlchemy: `inspect(obj).attrs`).
- `ObjectChangeHandler`: `created(obj)`, `updated(obj, changed_fields, change_set)`, `deleted(obj)`, `*_events()` до записи,
  `resolve(obj, rule_event)`, `renamed(obj, change_set, previous=None)` — старые URL как `deleted` (A21). Никогда не бросает:
  ошибка декларации — `error` в лог, пустой список.
- `Condition`/`FieldCondition`/`Equals("status", "published")` — как core 0.8 PHP.

### 3.5. Резолв URL

`RuleUrlResolver(rules, extractor, router: RouteUrlResolver | None, locator, config, logger)`: `route` → `router.generate(name, params,
locale, host)` (адаптер: Django `reverse`, FastAPI `url_path_for`, Flask `url_for`), корень хоста — `RouteOrigin.root(config, host)`
(`base_url_for(host)` иначе `https://<host>`), относительный результат ребейзится; `locales: "all"` при пустом списке локалей —
одна URL и предупреждение один раз на процесс (урок спеки 19 §2.8); `locale_hosts`; `via` — `ViaWalk` с `max_via_depth`/`max_via_fanout`;
`url`/`urls`/`resolver` (id из `ResolverLocator` или callable). `GuardedUrlResolver` — единственная точка «объект → URL» для
фасада и хуков. `explain(obj, event) -> list[ResolvedUrl]` с `rule`, `event`, `locale`.

### 3.6. Submitter, дебаунс, throttle, retry

Порядок `Submitter.submit(urls)` — спека 02: нормализация (`InvalidUrlError` → `skipped/invalid_url`, warning), `strict_hosts`/карта
ключей (`no_key`), `enabled`/`dry_run` (`disabled`/`dry_run`, лог по уровню `logging_levels`), дебаунс (`debounced`, чтение/запись
стора fail-open с warning), группировка и чанки, throttle, `client.submit_all`, `mark_submitted` только для ok/pending, listeners
(`add_listener(fn: Callable[[Result], None])`, изолированы), `SubmissionStore.record()` (never throws наружу). Часы — `Clock`
protocol, `SystemClock` (`time.time`), `FrozenClock` в тестах. `RetryPolicy.delay_after(results, attempt)` (60 с после 429 без
`Retry-After`, 5 с после 5xx/сети, ×2, потолок 3600), `RetryingSubmitter(inner, policy, sleeper=time.sleep)`, `WorkerOutcome`
(retryable/final, три строки лога) — для Celery/RQ/`django.tasks`.

### 3.7. Коллектор и единица работы

`Collector` держит `ContextVar[set[str] | None]`: `collecting()` открывает scope (Token, reset в `finally`); `collect(urls)` без
scope — сразу диспетчер (management command, Celery-задача); дедуп по нормализованному URL; `collector_max_urls` — ранний flush;
`reset()` — warning о потерянных; `detect_leaks` — `atexit`-проверка. `IndexNowKit.flush()` — `drain()` → `dispatcher.dispatch()`.
`wsgi.CollectorMiddleware(app, kit)`: открывает scope, оборачивает итерируемое тело ответа и вызывает `flush()` в `close()`
итератора (PEP 3333: сервер зовёт `close()` после отдачи тела — «после ответа», H06) — это же место, где Django шлёт
`request_finished` (§21). `asgi.CollectorMiddleware`: scope на `http`-scope, flush после `http.response.body` с `more_body: False`
(после `await app(...)` — ответ уже отправлен).

### 3.8. Диспетчеры

| Режим `dispatch` | Класс | Где уместен |
|---|---|---|
| `sync` (дефолт core) | `SyncDispatcher` | после ответа (WSGI `close()`, ASGI после body) — воркер занят ≤ `http_timeout`; CLI, задачи |
| `none` | `NullDispatcher` | собирать, не слать |
| `thread` | `ThreadDispatcher` | один daemon-поток с `queue.Queue`, `atexit` ждёт ≤ 5 с и пишет warning с числом потерянных URL; для сред без очереди, где sync-задержка неприемлема |
| `asyncio` | `AsyncTaskDispatcher` | ASGI без очереди, opt-in: `loop.create_task(kit.asubmit(urls))`, строгие ссылки в `set`, `add_done_callback(discard)` (факт §1.2); при остановке сервера задачи режутся — под ASGI дефолт `sync` = `await asubmit()` в middleware после ответа (спека 23 §3, 26 §9.11) |
| `callable` | `CallableDispatcher(fn)` | `fn(urls: list[str]) -> None` — Celery `task.delay`, RQ `queue.enqueue`, Dramatiq `send`, `django.tasks` `task.enqueue` (аргументы JSON-сериализуемы — список строк подходит) |
| адаптерные | `tasks` (Django 6.0+), `celery`? — нет, через `callable` | §21 |

`DispatcherFactory.from_config(config, submitter, logger, queue_factory=None)`: режим вне `{sync, none, thread, asyncio}` без фабрики
→ `ConfigurationError` «needs a queue dispatcher». `BatchingDispatcher(fn, batch_size, logger)` — чанки и `job_id` для очередей.

### 3.9. Adapter kit

- **Слой 1 — фабрики-функции** (`http.transport_from_config(config, client_locator=None)`, `debounce.store_from_config(config, cache_locator=None, default="memory")`,
  `dispatch.dispatcher_from_config(...)`, `Collector.from_config`, `TokenBucket.from_config`, `RuleUrlResolver.from_config`,
  `KeyFileResponder.from_config`) с едиными текстами ошибок; `IndexNowKit.create(config, *, transport=…, keys=…, normalizer=…,
  throttle=…, debounce=…, submitter=…, dispatcher=…, collector=…, resolver=…, reader=…, router=…, locator=…, extractor=…,
  logger=…, clock=…, failure_cache=…, submission_store=…, events=…)` — только именованные, несовместимая комбинация
  (`submitter` + `transport/debounce/throttle/normalizer`) отвергается.
- **Слой 2 — `ServicesBuilder(config, logger)` → `Services`** (спека 16 §3.2): сеттеры узлов принимают объект или
  `Callable[[Services], T]`; `Services` — `functools.cached_property` на каждый узел (`transport`, `keys`, `normalizer`, `throttle`,
  `debounce_store`, `client`, `submitter`, `collector`, `dispatcher`, `reader`, `rules`, `router`, `resolver_locator`, `url_resolver`,
  `guarded_resolver`, `changes`, `kit`, `key_file_responder`, `checker`, `submitter_factory`, `param_extractor`, `failure_cache`,
  `submission_store`, `clock`), `require_router()`, `has_collected()`, `flush_if_collected()`; никакого IO в `build()`; тест паритета
  слоёв (`ServicesParityTest`) — переносится.
- **`ConfigFactory`** — как PHP (§1.3), `load()` never throws: `critical` + `disabled()`.
- **`ObserverHelper.for_changes(changes, sink, logger)`** — `guard(obj, fn)`, `deliver(urls)`, `remember_deletion(obj, urls)` (в
  `weakref.WeakKeyDictionary`; объекты без `__weakref__`/хэшируемости — по `id()` с явным `take`), `take_deletion(obj)`.
- **`OptionalPackage` не переносится**: sitemap/verify/history — модули core; их «предикат» — `SitemapConfig.enabled` и т. п.;
  строки `check` `sitemap.enabled`/`verify.installed`→`verify.enabled`/`history.store` остаются (коды те же, где есть; `<feature>.installed`
  исчезает — записать в `docs/check-codes.md` Python как «не применяется»). Адаптерам не нужен «stub command».

### 3.10. Ключ-файл

`KeyFileResponder(keys, enabled)`: `body_for_path(path, host)` / `body_for_key(key, host)` → `str | None`; заголовки —
`config.key_file_headers()` (`Content-Type: text/plain; charset=utf-8`, `Cache-Control: public, max-age=<n>`, `Vary: Host` при
`hosts` или `strict_hosts`), `previous_key` отдаётся во время ротации. `wsgi.KeyFileApp(responder, config)` — вызываемое
`(environ, start_response)`, host из `HTTP_HOST` (без порта), 404 иначе; `asgi.KeyFileApp` — `async (scope, receive, send)`; оба —
и как middleware (`KeyFileApp(responder, config, next_app)`). Адаптеры: Django — view над `body_for_key` (§21), FastAPI/Starlette —
маршрут на `asgi.KeyFileApp` (§23), Flask — view (§24).

### 3.11. `check`, коды, `--json`

`Checker(config, keys, transport, checks)` — те же строки и коды, что `core/docs/check-codes.md` PHP: `config.*`, `environment.*`,
`key.*`, `key_file.*` (GET `<host>/<key>.txt` без редиректов: статус, тело, `Content-Type`, `Cache-Control`/`Age`, `robots.txt`,
`previous`), `probe.*` (`--live`), `debounce.store`, `check.failed`; Python-специфичное: `python.expat` (§7.5), `dispatch.thread`
(warning: URL теряются при выходе процесса). `check --json` валиден по `check.schema.json` — файл копируется в `docs/spec` как
кросс-языковой контракт (спека 26 §9.7). `--strict`, `--host`, `--probe-url`, `--sample`, `--sample-class` — как в PHP.

### 3.12. CLI без фреймворка

`python -m indexnowkit <cmd>` и console-script **`indexnowkit`** (не `indexnow`: на одном хосте может стоять PHP-бинарник
`indexnow` спеки 19b — коллизия в `PATH`; спека 26 §9.6). Команды — те же восемь, что у `indexnowkit/cli` (`check`, `config`,
`submit`, `sitemap`, `key generate`, `key file`, `history`, `status`; подкоманды `key generate`/`key file` вместо `key:generate` —
двоеточие в argparse-подкомандах неудобно), плюс `explain`/`submit-objects` нет (нет ORM). Глобальные опции `--env-file`,
`--no-env-file`, `--config` (JSON или TOML по расширению), `--state`, `-v`. Приоритет: опции > окружение > `.env` > `--config` >
умолчания. Файл состояния `.indexnow/state.sqlite` (0700; таблицы `indexnow_cache`, `indexnow_submissions`, `indexnow_sitemap_seen`);
умолчания CLI: `debounce.store = state`, `history.store = sqlite` над тем же файлом. `uvx indexnowkit check` — запуск без установки
(uv). Cron-строка в README — как у PHP CLI.

### 3.13. Sitemap, verify, history

Поведение и опции — как у PHP-пакетов (`sitemap/docs/*`, `verify/docs/configuration.md`, `history/docs/*`): `sitemap [url|file]
[--new-only] [--changed-since] [--dry-run] [--no-verify] [--json]`, полный прогон больше батча — предупреждение, `--new-only`
без стора — exit 2 с текстом; `verify.enabled` — декоратор сабмиттера со своим транспортом без редиректов (не `http.client`),
`robots.txt` через `RobotFileParser` (кэш в сторе дебаунса, TTL), `noindex`/canonical через `html.parser` на первых 256 КиБ,
404/410 — удаление; `history.store = sqlite|<адаптер>`, команды `history [--host --status --since --json --purge]`, `status --json`
по `status.schema.json` PHP (копия в spec).

### 3.14. Логирование и метрики

`logging.getLogger("indexnowkit")` с детьми (`indexnowkit.client`, `.submitter`, `.resolver`, `.hooks`, `.dispatch`) — имя
логгера и есть канал, префикс `indexnow: ` в текстах **не нужен** (спека 26 §1). Тексты — таблица `core/docs/operations.md` PHP
один к одному (`{engine} accepted {count} URL(s) for {host}`, …), уровни по `Config.LOG_EVENTS` и `logging.levels`;
`extra={"indexnow": {...}}` со структурными полями (`engine`, `host`, `count`, `reason`, `http_code`) для JSON-логгеров.
Метрики — `Result.metric_labels()`; хук `add_listener` для Prometheus/StatsD пользователя.

### 3.15. Async

`IndexNowKit.asubmit(urls)`: с `[httpx]` — `AsyncHttpxTransport` и `Submitter.asubmit()` (тот же конвейер, `await transport.apost`,
`await throttle.aacquire`; дебаунс-стор синхронный — sqlite/память быстрые, Django cache — sync API, документируется);
без httpx — `await asyncio.to_thread(self.submit, urls)`. `AsyncTaskDispatcher` — §3.8. Ничего async в правилах и резолве:
ORM-хуки SQLAlchemy async идут через sync-сессию в greenlet (спека 22).

### 3.16. Типизация и стиль

`py.typed`; `mypy --strict` на `src` и тестах; `ruff` (`E,F,I,UP,B,S,N,RUF`, формат ruff); PEP 604, `Self`, `StrEnum`; **без PEP 695**
до подъёма минимума на 3.12 (спека 26 §9.2); `TypeVar` по-старому; `Protocol` с `@runtime_checkable` только там, где нужен
`isinstance` (`StreamingTransport`). Публичное — `__all__` каждого модуля; `@internal` → `_`-префикс модулей (`config/_parser.py`).

## 4. BC и версии

`indexnowkit` 0.1.0; SemVer, до 1.0 минор может ломать (как `core/docs/bc.md`: три тира — Call / Implement (Protocol) / May grow;
enum'ы `Reason`/`Engine`/`RuleSource` растут минором; `Event`/`ResultStatus`/`CheckLevel` закрыты; именованные аргументы
`create()` и конструкторов — часть обещания; тексты логов и исключений — нет). `docs/bc.md` пакета — с 0.1.0. Extras: `httpx`
(`httpx>=0.28`), `testing` (`pytest>=8`). Минимальный Python поднимается в первом миноре после выхода предыдущего из
security-поддержки (3.11 → 3.12 после 2027-10; спека 17 §7) — `docs/compatibility.md`.

## 5. Тесты

- `tests/unit/*` по модулям; `tests/conformance/test_core.py` — C01–C22 (C13 через `RetryingSubmitter`, C22 через `FrozenClock`);
  `tests/conformance/test_ids.py` — сканирует имена `test_c\d\d`, `test_a…`, `test_s…`, `test_h…` в `src/indexnowkit/testing/conformance`
  и `tests/` пакетов workspace (аналог `ConformanceIdsTest`: каждый id ровно один раз, диапазон заморожен; адаптеры — по
  `pyproject.toml` с `indexnowkit` в `dependencies` и каталогом `tests/` — обязаны нести H01–H06).
- `tests/integration/test_urllib_transport.py` — через `MockIndexNowServer` в процессе (порт 0), сценарии `ok200 … timeout`
  (`timeout` — сервер спит 2 с при `http_timeout=0.5`); `test_httpx_transport.py` под `pytest.importorskip("httpx")`.
- Тесты README: `ReadmeAssertions.assert_ai_notes(dir, commands, option_keys)` для EN и RU; quickstart-фикстуры `tests/readme/`
  как `ReadmeQuickstartTest`.
- Мок-сервер: контрактный тест сценариев (те же имена и коды, что `router.php`) — таблица в спеке 03; парити-тест между PHP и
  Python не автоматизируется (разные репо), контракт — текст спеки.
- Матрица CI: Python 3.11, 3.12, 3.13, 3.14 (3.15 — `experimental: true` до GA 2026-10-01); `lowest` — `uv sync --resolution
  lowest-direct`; coverage-floor (`tests/coverage-floor.txt`, `coverage.py` + ratchet-скрипт как `bin/coverage-floor`);
  mutation (`mutmut`) — не в этой волне; вместо него **`hypothesis`** (dev-зависимость): `tests/property/test_normalizer.py`
  (идемпотентность, punycode round-trip, tracking-параметры, `sort_query`, длина ≤ `max_url_length`) и `test_config_parsing.py`
  (`from_mapping(to_mapping(c)) == c`, `from_mapping(mapping_from_env(env)) == from_env(env)`, булевы литералы, `INDEXNOW_HOSTS`)
  — спека 26 §9.9.

## 6. Документация

`README.md` + `README.ru.md` по шаблону спеки 90 (Who gets notified; Install `pip install indexnowkit` / `uv add`; Quick start с
`Config.from_env()` и `@indexnow`; `check`; How it works; CLI/cron; Limitations; Other packages; Notes for AI assistants;
Versioning), `docs/`: `configuration.md` (таблица опций + переменные окружения, генерируется скриптом из `Config.OPTIONS` как
`bin/config-table`), `rules.md` (аналог `attribute-reference.md`), `check-codes.md`, `operations.md`, `retries-and-queues.md`,
`submission-store.md`, `sitemap.md`, `verify.md`, `history.md`, `cli.md`, `state.md`, `adapters.md` («20-минутный адаптер» на
`ServicesBuilder`), `testing.md`, `bc.md`, `compatibility.md`. Docs-сайт — `indexnowkit.dev/python/` (спека 26 §6).

## 7. Риски и что проверить первым (research-first)

1. **`urllib.request` и HTTP/1.1 keep-alive** — `urlopen` открывает соединение на запрос; для 10 000 URL это один POST, для
   `sitemap` — до `max_sitemaps` GET. Приемлемо; `httpx` extra для пулов. Проверить при реализации: `Connection: close` по умолчанию.
2. **`ThreadDispatcher` и gunicorn `--preload`/fork** — поток создаётся лениво в воркере (после fork), не в мастере; тест с
   `os.fork()` не нужен, достаточно ленивого старта и документации.
3. **`contextvars` в потоках WSGI-сервера** — новый поток = пустой контекст: scope открывается middleware в потоке запроса; для
   `asyncio.to_thread` контекст копируется автоматически (3.9+). `ThreadDispatcher` не читает контекст (получает список).
4. **`AsyncTaskDispatcher` при остановке сервера** — незавершённые задачи отменяются; документировать как у `thread`, `check`
   предупреждает (`dispatch.asyncio`).
5. **Expat в системном Python** — `pyexpat.EXPAT_VERSION < 2.7.2` (факт §1.2): `check` пишет `python.expat` warning; `max_bytes`
   сохраняется. Проверить на образе `python:3.12-slim` при реализации.
6. **`html.parser` на битом HTML** — терпим к ошибкам, не бросает; лимит 256 КиБ; тест на обрезанный документ.
7. **`sqlite3` и несколько процессов cron** — WAL + `busy_timeout=5000`; тест на два соединения.
8. **`from_env` булевы** — `INDEXNOW_DRY_RUN=false` обязан быть False (урок W1); тест на все восемь литералов.
9. **Starlette 1.x** — сайт не резолвился 2026-09-10; release notes прочитать перед §23 (что сломано в 1.0: `Request.state`? middleware
   ABI?). До этого `asgi.CollectorMiddleware` пишется по чистому ASGI-спеку (asgi.readthedocs.io), без Starlette.
10. **httpx 1.0** — `1.0.dev6` 2026-08-31; пин `httpx>=0.28,<2` и тест на dev при появлении rc.
11. **Имя console-script `indexnowkit`** — проверить, что PyPI-имя дистрибутива и script не конфликтуют с существующим бинарником
    в популярных образах (нет: 404 на PyPI, §1.4).

## 8. Не делать (рассмотрено)

- `requests`/`aiohttp` транспорты: `urllib` + `httpx` покрывают; `requests` — синхронный дубль без выгоды.
- `pydantic`-модель конфигурации: зависимость ради валидации, которую делает `__post_init__`; адаптер FastAPI может обернуть в
  `pydantic-settings` сам (§23).
- `click`/`typer` в core: `argparse` достаточно; Flask-адаптер оборачивает раннеры в click (§24).
- Docker-образ и zipapp: `uvx indexnowkit`, PHP-образ для «любого сайта».
- Отдельные дистрибутивы console/testing/sitemap/verify/history: нет зависимостей, которые бы это оправдали (спека 26 §9.1).
- Redis-стор дебаунса как extra: Django cache / любой объект с `get/set/add` (`CacheDebounceStore`) — 10 строк у пользователя.
- `defusedxml`: sitemap без DTD, лимит байтов, Expat ≥ 2.7.2 (§7.5).
- PEP 695 синтаксис до минимума 3.12.

## 9. `[решение]` по core (рекомендации; общие решения волны — спека 26 §9)

1. **Один дистрибутив с модулями sitemap/verify/history/cli/testing-extra** — да (§0).
2. **Console-script `indexnowkit`, не `indexnow`** — да (§3.12).
3. **Транспорт: `urllib` sync + `httpx` extra async; без `requests`** — да.
4. **Дебаунс-сторы: memory, sqlite, `CacheDebounceStore` над `get/set/add`; без redis-extra** — да.
5. **Accessor в правилах: строка или callable; в `fields`/`when_fields` — только строки** — да.
6. **Логгер `indexnowkit.*` без префикса `indexnow: ` в тексте** — да (тексты иначе один к одному с PHP).
7. **`dispatch` в core: `sync` (дефолт), `none`, `thread`, `asyncio`, `callable`** — да; адаптерные режимы — в адаптерах.
8. **CLI-подкоманды `key generate` / `key file`** (не `key:generate`) — да.

## 10. Definition of Done

- `pip install indexnowkit` в чистом venv на 3.11–3.14: `python -m indexnowkit check` с `INDEXNOW_KEY`, `INDEXNOW_BASE_URL` против
  `MockIndexNowServer` зелёный; `indexnowkit key generate --write-env && indexnowkit key file /tmp/docroot && indexnowkit check --live`;
  `sitemap --new-only` дважды — второй раз «0 new or changed».
- C01–C22 зелёные; `test_ids` видит ровно C01–C22, A01–A21 (+b/c), S01–S08 в ките; `mypy --strict` и `ruff` чистые; coverage-floor
  записан с CI; `uv build --no-sources` собирает sdist+wheel; README EN/RU с AI-notes проходят `ReadmeAssertions`.
- `check --json` валиден по `check.schema.json` из `docs/spec`; `status --json` — по `status.schema.json`.
- Docs пакета §6 написаны; `docs/compatibility.md` с матрицей Python × EOL.
