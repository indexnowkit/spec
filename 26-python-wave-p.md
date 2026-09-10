# 26. Волна P: Python-семейство — аудит переноса, порядок, гейт, инфраструктура

Статус: **спецификация волны, 2026-09-10; решения §9 приняты той же датой** (шаг 1 волны P — аудит и переписывание спек 20–25;
код — после «действуй» по execution-промпту). Волна делится на P1 (core, django, sqlalchemy, fastapi → PyPI) и P2 (wagtail, flask). Основание: PHP-семейство выпущено целиком (core 0.13.1, console 0.5.0, testing 0.3.2, sitemap 0.9.0, verify 0.4.0,
history 0.4.0, doctrine 0.9.0, symfony-bundle 0.15.0, laravel 0.15.0, yii2 0.14.0, yii3 0.1.0, cli 0.1.0 — спека 17 §16, 19b), домен
`indexnowkit.dev` с docs PHP под `/php/`, дистрибуция спеки 90 начата. Образец формата — спека 19b. Спеки 20–25 переписаны в
этом формате той же датой; здесь — то, что относится к волне в целом.

## 0. Цель и границы

Python-семейство с тем же поведением, той же схемой конфигурации, теми же кодами `check` и теми же идентификаторами conformance,
что у PHP, в идиомах Python, за одну волну из шести шагов: `indexnowkit` (core со всем внутри) → `indexnowkit-django` →
`indexnowkit-sqlalchemy` → `indexnowkit-fastapi` → `indexnowkit-flask` (по решению) → `indexnowkit-wagtail` (по решению). Ниша
Django/SQLAlchemy/FastAPI на PyPI пуста (спека 20 §1.4: три пакета-клиента без commit-safety, батчей и `check`).

Границы: без JS/Bitrix (свои волны); без Docker-образа Python, zipapp, `django-cms`, `Quart`, `Litestar` (рецепты на core); 1.0 не
предлагается (память `feedback-no-1-0-until-stable`).

## 1. Аудит переноса: каждая возможность PHP-семейства → Python

Легенда: **как есть** — та же семантика и имена (snake_case); **иначе** — та же цель, другой механизм; **нет** — не переносится, причина.

| PHP-возможность (где) | Python | Как / почему |
|---|---|---|
| Протокол: POST всегда, GET как опция, батчи ≤ 10 000, разрез по host, `engines` × host, `engine_aliases`, `hosts.<host>.engines` (спека 01, `Client`) | как есть | `indexnowkit.client`, `Engine(StrEnum)` |
| `Result`/`Reason` (16)/`ResultStatus`, `metric_labels()` без host, `retryable_urls()` | как есть | frozen dataclass + `StrEnum` |
| `Config::OPTIONS` (45 ключей без `serve_key_file`), проверки конструктора, тексты «факт — что можно — как починить» | как есть | `Config.__post_init__`, `OPTIONS` |
| `strict_hosts`, `previous_key`, `key_location`, карта `hosts` с `{key, key_location, base_url, engines, previous_key}`, `production_environments`, страховка `dry_run` + `dryRunExplicit` | как есть, **с 0.1.0** | спека 20 §1.3 |
| `Config::fromEnv()`, `arrayFromEnv()` «только заданное», правило `INDEXNOW_<BLOCK>_<KEY>` для блоков (19b §3.3) | как есть, с 0.1.0 | `from_env`, `mapping_from_env` покрывает ядро и блоки сразу |
| Булевы разбираются, не кастуются (аудит 0.13 W1) | как есть | восемь литералов, пустая строка = не задано |
| `Adapter\ConfigFactory` (`build`/`load`/`unknownOptions`/`disabled`, `ownedOptions`, `dispatchModes`, `autoDispatch`, `needBaseUrl`, `defaults`, `validate`, `checkCommand`, `ignoreBlocks`) | как есть | `indexnowkit.adapter.ConfigFactory` |
| `Adapter\OptionalPackage`, стабы `*NotInstalledCommand`, `optional-packages-absent` CI-джоба | **нет** | sitemap/verify/history — модули одного дистрибутива; предикат — `<block>.enabled`; коды `<feature>.installed` не применяются (записать в `check-codes.md`) |
| `Adapter\ServicesBuilder`/`Services` (слой 2), паритет со слоем 1 | как есть | `cached_property`-узлы, тест паритета |
| Статические фабрики слоя 1 (`TransportFactory::lazy`, `DebounceStoreFactory`, `DispatcherFactory`, `*::fromConfig`) | как есть | функции модулей |
| `IndexNowKit::create()` только именованные, запрет `submitter` + `transport/...` | как есть | keyword-only |
| `Key\*`: генератор (CSPRNG hex 32), валидатор+маска, `StaticKeyProvider`, `KeyFileResponder`, заголовки (`Vary: Host`) | как есть | `secrets.token_hex` |
| PSR-15 `KeyFileRequestHandler` (18 §3.5) | иначе | `wsgi.KeyFileApp` и `asgi.KeyFileApp` — те же роли (маршрут и middleware) на стандартных интерфейсах Python |
| Модель правил: `UrlRule`, повторяемый `#[IndexNow]`, `#[IndexNowDefaults]`, `#[IndexNowUrl]`, `ParamValue` ×4, `when` конъюнктивно, `fields`/`whenFields`, `events`, `locales`, `host`, `via` с лимитами, наследование по `name`, `RuleRegistry` | как есть | декораторы `@indexnow`/`@indexnow_defaults`/`@indexnow_url`; accessor — строка или callable (спека 20 §9.5) |
| `ChangeClassifier` (таблица 02), геттер → поле по конвенции | как есть | `is_x`/`has_x`/`x` |
| `ParamExtractor` + `SubjectReaderInterface`, `plain()` | как есть | `getattr` + точечные пути; ридеры адаптеров |
| `ObjectChangeHandler` (`*Events()` до записи, `renamed()`, never-throw), `GuardedUrlResolver`, `explain()` | как есть | |
| `RouteUrlResolverInterface` ×4, `Url\RouteOrigin` | как есть | Django `reverse`, FastAPI `url_path_for`, Flask `url_for`; SQLAlchemy — без роутера |
| `Url\UrlNormalizer` + `CanonicalUrlNormalizer` (`normalizer.*`), Punycode | как есть | `idna`-кодек stdlib |
| `Collector` (request-scoped, дедуп, `max_urls`, `detect_leaks`, `reset()` с warning) | как есть | `contextvars` |
| `Transaction\TransactionStaging`/`StagingFrame`, `VerifyingStaging` | иначе | Django — `on_commit` нативно (staging не нужен); SQLAlchemy — `SessionStaging` по кадрам `after_transaction_create/end`; `VerifyingStaging` не нужен (у обоих есть сигнал commit) |
| Диспетчеры `sync`/`none`/`Callable`/`BatchingDispatcher`; `RetryPolicy`, `RetryingSubmitter`, `WorkerOutcome` | как есть + `thread`, `asyncio` | Python-среды без очереди |
| Очереди: Messenger/Laravel Queue/yii2-queue джобы | иначе | Django `tasks` (6.0 `django.tasks`, бэкпорт `django-tasks`), `callable` для Celery/RQ/Dramatiq/huey/arq/taskiq |
| Дебаунс: `Memory` (bounded), `Null`, `Psr16` | иначе | `Memory`, `Null`, `Sqlite` (файл состояния — в core, не в CLI-пакете), `CacheDebounceStore` над `get/set/add` (Django cache, Flask-Caching, redis-py-подобные) |
| `Throttle\TokenBucket` с `ClockInterface`, `FrozenClock` | как есть | + `aacquire` |
| `ForbiddenCounter` в общем кэше, `critical` раз на флот | как есть | тот же стор |
| `SubmissionStoreInterface`, `NullSubmissionStore`, `SubmissionRecord`; history `psr16`/`pdo` сторы, S01–S08 | иначе | core: `Null`, `Sqlite`; Django: модель + миграция; `psr16`-кольцо — нет (кэш-стор в Python-мире — не место для истории; `sqlite` дешевле) |
| `indexnowkit/sitemap`: ридер (index, gzip, text, `lastmod`, spool, лимиты), `--changed-since`, `--new-only` + `SeenStoreInterface`, `SitemapSpoolCheck` | как есть, модуль core | `iterparse`; `SqliteSeenStore` в core; Django — `--from-django-sitemaps` |
| `indexnowkit/verify`: pre-flight (noindex, robots, canonical, redirects, origin errors), свой транспорт без редиректов, `check --sample`, `max_batch`, `time_budget`, `robots_cache_ttl` | как есть, модуль core | `html.parser`, `urllib.robotparser` |
| `check`: коды (`core/docs/check-codes.md`), `--json` (`check.schema.json`), `--strict`, `--live`, `--host`, `--probe-url`, `--sample` | как есть | схема — в `docs/spec` как кросс-языковой контракт (§9.7); новые коды `python.expat`, `dispatch.thread`, `dispatch.asyncio` |
| Команды ×9 (+`key:file`), `Definitions` один раз, `Vocabulary`, раннеры, `ResultRenderer`, `ExitCode`, `ConfigSourceInterface` | как есть | `argparse`-`Definitions` с конвертерами в Django `add_arguments` и click; имена: CLI `indexnowkit <cmd>`, Django `indexnow_<cmd>`, Flask `flask indexnow <cmd>` |
| `indexnowkit/cli`: `.env`, JSON-конфиг, файл состояния, `key:file`, умолчания `state`/`pdo` | как есть, модуль core | `indexnowkit.cli`; JSON **и** TOML (`tomllib`) |
| PHAR, Docker-образ, GitHub Action (19b) | **нет** | `uvx indexnowkit` — путь без установки; PHP-образ и Action покрывают «любой сайт»; образ Python — по спросу |
| Логи: префикс `indexnow: `, тексты `operations.md`, уровни `LOG_EVENTS`, маскирование ключей | иначе (префикс) / как есть (тексты) | логгер `indexnowkit.*` — имя и есть канал; `extra` со структурой |
| PSR-14 события `Submitter::addListener()` | как есть | `add_listener(callable)`; Django — сигналы поверх |
| Тест-двойники `FakeTransport`, `ArrayLogger`, `FrozenClock`, `RecordingDispatcher` | как есть, кроме `ArrayLogger` | `caplog` pytest |
| Conformance-киты: `CoreConformanceTestCase` (11 из C), `OrmConformanceTestCase` (A01–A21 + b/c), `SubmissionStoreConformanceTestCase` (S01–S08), `KeyFileAssertions`/`CheckOutputAssertions` (H01–H06), `ReadmeAssertions`, `OptionalPackageAssertions`, `ConformanceIdsTest` | как есть | pytest-классы-миксины с абстрактными фикстурами; `test_ids` по именам `test_c01_*`; `OptionalPackageAssertions` → `OptionalFeatureAssertions` (блок выключен) |
| Mock-сервер `router.php` (сценарии, `/_mock/requests`, `MOCK_KEYS`, `/large-document.xml[.gz]`) | иначе | `MockIndexNowServer` на `http.server` в процессе, тот же контракт — спека 03 обновлена: контракт — текст спеки, реализация на язык |
| Spec 03 «fixtures как YAML, раннеры читают YAML» | **нет** (исправление спеки 03) | в PHP YAML не было; абстрактные тест-кейсы — норма семейства |
| README-шаблон 90, RU-версия, «Notes for AI assistants» + тест, `llms.txt`, docs-сайт MkDocs (`bin/docs-collect`) | как есть | `bin/docs-collect` — Python-скрипт, переиспользуется с `BASE_URL=/python/` |
| `bc.md` три тира, `compatibility.md`, правило подъёма минимальной версии | как есть | Python EOL-таблица |
| CI: матрица версий, `lowest`, coverage-floor, mutation (Infection), taint (Psalm), `composer validate`, `config-table --check` | иначе | `uv`, `ruff`, `mypy --strict`, `pytest`, coverage-floor (тот же ratchet), `uv build --no-sources`, генератор таблицы конфигурации `--check`; mutation (`mutmut`) — позже (§9.9); taint — `ruff S` (flake8-bandit), полного аналога нет |
| Монорепо + split-репозитории + Packagist-webhook, `bin/tag`, `packagist-wait` | иначе | один репозиторий `indexnowkit/python` без сплитов (PyPI не читает VCS), теги `<pkg>@<ver>`, trusted publishing (§6) |
| Профилер Symfony, `about` Laravel | нет | по спросу (debug-toolbar) |
| `verify-on-commit` Yii (`VerifyingStaging`) | нет | нет фреймворка без сигнала commit |

## 2. Принципы волны

1. **Поведение — PHP, идиомы — Python.** Спорное решается в пользу совпадения имён ключей, кодов, идентификаторов тестов и текстов
   логов; форма кода — по PEP 8/typing/dataclass/Protocol.
2. **Один дистрибутив core, тонкие адаптеры.** Ничего не режется по пакетам без причины-зависимости.
3. **С первой версии — весь опыт PHP** (спека 20 §1.3): никаких «потом добавим `strict_hosts`».
4. **Research-first остаётся**: риски §7 спек 20–25 проверяются unit-тестом или чтением документации **до** дизайна кода.
5. **Гейт до пуша, версии с CI** (уроки волн M/N: coverage-floor записывать числом CI; PHP 8.5 ↔ здесь Python 3.15).
6. **Тексты — «факт — что можно — как починить»; ключи маскируются; ничего из хука не бросает.**

## 3. Порядок и объём

| Шаг | Пакет | Что | Оценка (дни) |
|---|---|---|---|
| 0 | инфраструктура | `python/` в этом монорепо, uv workspace, `pyproject` корня, `bin/` (Docker-обёртки над `python:3.12-slim` + uv, как `php/bin`), CI, docs-site, репо `indexnowkit/python` (§6) | 1–2 |
| 1 | `indexnowkit` 0.1.0 | спека 20 целиком: протокол, правила, резолв, коллектор, диспетчеры, дебаунс, retry, `check`, CLI + состояние, sitemap, verify, history, wsgi/asgi, testing-extra (киты + мок), README EN/RU, docs | 7–10 |
| 2 | `indexnowkit-django` 0.1.0 | спека 21: сигналы + `on_commit`, миксин `from_db`, команды ×9, checks, key-view, tasks/callable, история-модель, sitemaps | 4–6 |
| 3 | `indexnowkit-sqlalchemy` 0.1.0 | спека 22: три+два события, `SessionStaging`, async | 2–3 |
| 4 | `indexnowkit-fastapi` 0.1.0 | спека 23 | 1–2 |
| — | **релиз P1** | теги core, django, sqlalchemy (вместе, после зелёных 2 и 3 — §9.4), fastapi; PyPI; дистрибуция (спека 90): метаданные, djangopackages, awesome-django PR, Django forum Show & Tell, Habr | 1 (+ пользователь) |
| 5 (P2) | `indexnowkit-wagtail` 0.1.0 | спека 25; Wagtail packages | 1–2 |
| 6 (P2) | `indexnowkit-flask` 0.1.0 | спека 24 | 1–2 |

Итого 17–27 дней при темпе PHP-волн; P1 (шаги 0–4) — до первой публикации, P2 (5–6) — после (§9.5).

## 4. Версии и совместимость

- Все пакеты **0.1.0**; SemVer, до 1.0 минор может ломать; `bc.md` на пакет с тремя тирами (Call / Protocol-Implement / May grow).
- **Python ≥ 3.11** для всего семейства (§9.2): 3.10 EOL 2026-10; 3.11 — до 2027-10; uv-workspace требует общего минимума; PEP 695
  не используется до подъёма на 3.12 (первый минор после 2027-10 по правилу спеки 17 §7). Матрица 3.11–3.14, 3.15 `experimental`.
- Django `>=5.2,<6.2` (5.2 LTS до 2028-04, 6.0, 6.1); SQLAlchemy `>=2.0,<2.2`; FastAPI `>=0.110`; Flask `>=3.0`; Wagtail `>=6.3`(?)
  — минимум по LTS-политике Wagtail (спека 25 §7.1).
- Зависимости между пакетами: `indexnowkit>=0.1,<0.2` (каскад при 0.2 — как в PHP, но пакетов пять, не двенадцать).
- `docs/compatibility.md` в core: Python × EOL, фреймворк × EOL, правило подъёма.

## 5. Гейт (перед каждым пушем; в CI — обязательные джобы)

1. `uv lock --check`; `uv sync --all-packages`; `ruff check` + `ruff format --check`; `mypy --strict` по `src` и `tests` всех членов.
2. `pytest` по матрице: core 3.11–3.14 (+3.15 experimental), lowest-direct на 3.11; django × Django 5.2/6.0/6.1; sqlalchemy × 2.0/2.1rc;
   fastapi/flask/wagtail × latest + lowest. PostgreSQL job для django (история) и sqlalchemy (savepoint-семантика).
3. Conformance: `test_ids` — C01–C22 (core), A01–A21+b/c (кит и каждый ORM-адаптер), S01–S08 (каждый стор), H01–H06 (каждый
   фреймворк-адаптер, читается с файловой системы workspace, как в PHP); ни одного пропуска без записи в README.
4. `ReadmeAssertions` EN+RU для каждого пакета; quickstart-фикстуры README компилируются и проходят (`tests/readme/`).
5. Coverage-floor (`tests/coverage-floor.txt`, ratchet) — с первой CI-джобы, числом CI.
6. `uv build --no-sources --package <pkg>` для каждого; `twine check`? — `uv publish --check-url`/`--dry-run`? нет такого: проверка
   метаданных — `python -m twine check dist/*` в CI (twine как dev-инструмент) или `uv build` + `pip install dist/*.whl` в чистом venv
   (smoke: `python -c "import indexnowkit"`).
7. Генератор таблицы конфигурации `--check` (docs синхронны с `Config.OPTIONS`); `mkdocs build --strict`.
8. `check --json`/`status --json` валидны по схемам из `docs/spec` (`jsonschema` в dev-зависимостях).

## 6. Инфраструктура

- **Монорепо**: `python/` рядом с `php/` в этом workspace; зеркало — репозиторий `indexnowkit/python` (subtree-push как `php/`,
  `git subtree split --prefix=python`); `python/README.md`, `AGENTS.md`, `CHANGELOG.md` семьи, `packages/<pkg>/CHANGELOG.md`.
  Раскладка: `python/pyproject.toml` (workspace root, dev-зависимости: pytest, pytest-django, pytest-asyncio, mypy, ruff, coverage,
  jsonschema, django-stubs, twine), `python/packages/{indexnowkit,indexnowkit-django,indexnowkit-sqlalchemy,indexnowkit-fastapi,
  indexnowkit-flask,indexnowkit-wagtail}/{pyproject.toml,src/,tests/,docs/,README.md,README.ru.md,CHANGELOG.md}`, `python/bin/`
  (`ci`, `test`, `lint`, `docs-collect`, `tag`, `coverage-floor`, `config-table`), корневой `bin/spec-sync --check` workspace
  (§9.7: `docs/spec/{check,status}.schema.json` = копии в `php/packages/{console,history}/docs/` = `python/packages/indexnowkit/docs/`),
  `python/docker/python/Dockerfile`
  (`python:3.12-slim` + uv; `PYTHON_VERSION` как `PHP_VERSION`), `python/docs-site/` (MkDocs Material, `BASE_URL=https://indexnowkit.dev/python/`).
- **Что делает `gh`** (агент): создать `indexnowkit/python` (public, issues on, wiki/projects off, topics `indexnow, seo, python,
  django, sqlalchemy, fastapi`), deploy-key для subtree-push (или push по SSH-ключу пользователя, как для `php`), Pages «GitHub
  Actions», секреты — не нужны (trusted publishing без токенов), environment `pypi` с required reviewers? — нет (одиночный
  мейнтейнер), просто `environment: pypi`.
- **Что делает пользователь**: аккаунт PyPI с 2FA — **сделано 2026-09-10** (username `somework`); **pending trusted publishers** —
  **сделано 2026-09-10 для трёх**: `indexnowkit`, `indexnowkit-django`, `indexnowkit-sqlalchemy` (owner `indexnowkit`, repository
  `python`, workflow `release.yml`). Два ограничения PyPI, найденные при регистрации: (1) **одна тройка repo/workflow/environment —
  один pending publisher** («matching this configuration has already been registered for a different project name»), поэтому
  environment — **на пакет**: `pypi-core`, `pypi-django`, `pypi-sqlalchemy`, `pypi-fastapi`, `pypi-flask`, `pypi-wagtail`;
  `release.yml` ставит `environment: pypi-${{ <короткое имя из тега> }}` (имя environment может быть выражением); шесть environments
  создаются в репозитории `gh api` на шаге 0; (2) **не больше трёх pending publishers одновременно** («You can't register more than 3
  pending trusted publishers at once») — publishers для fastapi/flask/wagtail добавляются после создания первых проектов (pending →
  ordinary, слот освобождается), то есть ровно между P1 и P2. Pending publisher **не резервирует имя** (текст страницы) — резерв
  даёт только первый релиз. Заявка на организацию `indexnowkit` (Community, URL `https://indexnowkit.dev`) — **подана 2026-09-10**,
  ручная модерация; пакеты переводятся в неё после публикации. Остаётся: аккаунт djangopackages.org (GitHub-логин) для регистрации
  пакетов и grid; Wagtail packages; посты (форум Django, Habr) — как в PHP-дистрибуции.
- **Релиз**: тег `<pkg>@<ver>` в монорепо → subtree-push в `indexnowkit/python` с тем же тегом → `release.yml` сплита: `uv build
  --package <pkg> --no-sources` → `pypa/gh-action-pypi-publish@release/v1` (`permissions: id-token: write`, `environment: pypi-<pkg>`) →
  GitHub release из секции CHANGELOG (`bin/release-notes`, как в PHP). Порядок тегов: core → django → sqlalchemy → fastapi → flask →
  wagtail; между ними — ждать индекс PyPI (`https://pypi.org/pypi/<pkg>/<ver>/json` → 200; обычно секунды, `bin/pypi-wait`).
- **Docs**: `docs.yml` в `indexnowkit/python` строит `docs-site` и деплоит в Pages проекта; org-сайт `indexnowkit.github.io` уже
  отдаёт `/php/` из проекта php — `/python/` появится тем же путём (память `project-indexnowkit-dev-domain`); ссылка с `/php/` и
  из `indexnowkit.github.io` индекса.
- **CI**: `ci.yml` — матрица §5 (`astral-sh/setup-uv`, `uv python install`), `docs.yml`, `release.yml`; Dependabot для
  GitHub Actions и pip (`uv.lock`); `SECURITY.md`, `CONTRIBUTING.md`, CoC — копии PHP.

## 7. Риски волны (сверх рисков в 20–25)

1. **uv workspace и разный `requires-python`** — все члены на `>=3.11`; Django-адаптер на 6.x всё равно требует 3.12 у пользователя
   (решается зависимостью Django, не нашим минимумом).
2. **Trusted publishing до первого релиза** — pending publisher обязан существовать до тега core 0.1.0; иначе `release.yml` падает
   на 403. Три первых зарегистрированы (§6); лимит «три одновременно» означает, что перед P2 надо дождаться, пока проекты P1
   созданы, и только потом заводить publishers fastapi/flask/wagtail — шаг чек-листа релиза P1.
3. **Тег в монорепо и subtree**: тот же урок 19b (первый тег в пустом split-репо не запускает workflow) — здесь один репозиторий
   `python` создаётся с `main` до первого тега.
4. **`python:3.12-slim` Expat** — версия Expat в Debian slim может быть < 2.7.2 → `check` предупреждает; тесты sitemap на bomb — с лимитом.
5. **Starlette 1.x / httpx 1.0 / SQLAlchemy 2.1 GA** — три мажора в пути: пины `<2` и `experimental`-джобы.
6. **Объём core (шаг 1)** — 7–10 дней одним пакетом; смягчение: релиз core 0.1.0 только после зелёного Django-адаптера (ORM-кит —
   единственная реальная проверка модели правил), то есть теги идут после шага 2, как в PHP core 0.1.0 шёл с бандлом.
7. **Имя `indexnowkit` на PyPI** — свободно 2026-09-10; регистрируется pending publisher'ом сразу (резервирует имя).

## 8. Не делать (рассмотрено)

- Split-репозитории на пакет (PyPI не нуждается; issues — в одном месте).
- Docker-образ/zipapp/Action для Python CLI (PHP покрывает).
- Дистрибутивы `indexnowkit-console/testing/sitemap/verify/history`.
- `pydantic`, `requests`, `click` в core; `django-model-utils`; `defusedxml`.
- Mutation testing в этой волне (§9.9); taint-аналог.
- Поддержка Python 3.10, Django 4.2/5.1, SQLAlchemy 1.4, Flask 2.
- PR в `wagtail-indexnow` (спека 25 §8).
- JS-волна, Bitrix — не начинать.

## 9. `[решение]` — приняты 2026-09-10 после адверсального прохода (пункты 4, 5, 7, 9, 11 изменены против первой рекомендации)

1. **Один дистрибутив `indexnowkit` с модулями sitemap/verify/history/cli и extra `[testing]`** — (a) да. Альтернатива (b) —
   зеркало PHP из шести дистрибутивов: цена — шесть релизов на каждое изменение и каскад версий, выгода — ноль (зависимостей нет).
2. **Минимальный Python 3.11** — (a) да. Альтернатива (b) 3.12: PEP 695/`override`/`batched`, но Debian 12 (3.11) и Django 5.2-на-3.11
   отсекаются на 13 месяцев раньше правила спеки 17 §7. Цена (a) — без PEP 695 до 2027-10.
3. **Один репозиторий `indexnowkit/python`, теги `<pkg>@<ver>`, trusted publishing, без сплитов** — да. Альтернатива — сплиты как в
   PHP: цена — deploy-keys ×6 и стадии; выгоды у PyPI нет.
4. **Порядок: core → django → sqlalchemy → fastapi → (P2) flask → wagtail; релиз core после зелёных Django *и* SQLAlchemy** —
   **принято 2026-09-10** (адверсальный проход). Django — post-write хуки (`post_save`), SQLAlchemy — pre-write события (`before_flush`):
   два уровня `ObjectChangeHandler` (`*_events()` до записи vs `created()` после); PHP core стабилизировался после той же пары
   Doctrine + Laravel. Цена: +2–3 дня до первого тега; теги core/django/sqlalchemy — вместе. Отвергнуто: релиз core сразу (ломающий
   0.2.0 через неделю), релиз после одного Django.
5. **Flask и Wagtail — да, оба, но как P2 после первой публикации** — **принято**: волна делится на P1 (core, django, sqlalchemy,
   fastapi → PyPI) и P2 (wagtail первым — есть спрос: 1 111 загрузок/мес у конкурента с 1★; затем flask). Ранний релиз важнее полноты;
   оба по 1–2 дня и ничего не блокируют. Отвергнуто: «всё в одной волне» (+3–4 дня до релиза) и «вынести насовсем».
6. **Console-script `indexnowkit`** (не `indexnow`) — да; `uvx indexnowkit` работает только при совпадении имени скрипта с именем
   дистрибутива (иначе `uvx --from indexnowkit indexnow`), плюс коллизия с PHP-бинарником на одном хосте. Альтернатива
   `indexnow-py` — хуже читается.
7. **`check.schema.json` и `status.schema.json` — канон в `docs/spec`, копии в `php/` и `python/`, проверка синхронности** —
   **принято** (не «копии в spec»: копия без проверки дрейфует). Канон — файлы `docs/spec/check.schema.json`, `docs/spec/status.schema.json`
   (положены этим шагом байт-идентично из `php/packages/console/docs/` и `php/packages/history/docs/`; `$id` в них пока указывает на
   `indexnowkit/php` — перевод `$id` на `indexnowkit/spec` во всех копиях разом делает шаг 0 вместе со скриптом); в workspace лежат и `docs/spec`, и `php/`, и
   будущий `python/` — скрипт `bin/spec-sync --check` (cmp трёх копий) в корне workspace, часть шага 0 (§6), гоняется перед коммитом.
8. **Mock-сервер — своя реализация на язык по контракту спеки 03** — да; альтернатива — Docker service container с `router.php`:
   Docker в каждом dev-цикле и в CI Python-репо ради 140 строк.
9. **Mutation testing — не в волне; вместо него `hypothesis`** — **принято**: property-тесты на `UrlNormalizer` (RFC 3986, punycode,
   tracking-параметры, идемпотентность `normalize(normalize(x)) == normalize(x)`) и на парсер `Config` (`from_mapping`/`mapping_from_env`
   round-trip, восемь булевых литералов, `INDEXNOW_HOSTS`-строки) — ~1 день внутри шага 1; нормализатор — единственное место, где
   mutation в PHP ловил бы реальное. `mutmut` — как в PHP, после стабилизации.
10. **Django settings `INDEXNOW = {...}` snake_case вложенно** — да (спека 21 §9.1; прецедент — вложенные lowercase-ключи `LOGGING`).
11. **`dispatch`: Django `auto` (tasks при production-бэкенде), core `sync`, FastAPI — `sync`, выполняемый как `await asubmit()` в
    middleware после отправки ответа** — **принято** (было: FastAPI `asyncio`). Отдельная задача loop режется при остановке uvicorn;
    `await` после ответа не блокирует loop (httpx async), держит только task запроса — graceful shutdown его ждёт. Без httpx —
    `to_thread`. `asyncio` остаётся opt-in (спека 23 §3, §9.2).
12. **Wagtail — свой пакет, не PR** — да (спека 25 §9.1).
13. **Инфраструктура `bin/` на Docker (`python:3.12-slim` + uv), Python локально не ставить** — да (правило сессии; воспроизводимость
    как у `php/bin`).

## 10. Definition of Done волны

- Шесть (или четыре, по §9.5) пакетов на PyPI с 0.1.0, trusted publishing, GitHub releases; `pip install indexnowkit-django` в чистом
  `startproject` — квикстарт README проходит внешним прогоном (как Yii2 в волне 0b).
- Гейт §5 зелёный на всей матрице; conformance-идентификаторы — полные; docs-сайт `indexnowkit.dev/python/` собран, `llms.txt` есть.
- Спеки 00/03/91/README обновлены (сделано этим шагом), спека 26 §1 — таблица без строк «решить позже».
- Дистрибуция: PyPI-метаданные (keywords, classifiers, `project.urls`), djangopackages (пакеты + grid «SEO»), awesome-django PR,
  Wagtail packages, пост на форуме Django / Habr — с чек-листом «пользователь/агент» как в `docs/plans/distribution-2026-09-10.md`.
- Память `project-wave-p-python` обновлена; execution-промпт волны P записан в `docs/plans/`.
