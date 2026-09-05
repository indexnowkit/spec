# 91. Roadmap и критерии готовности

## Фазы

| Фаза | Что | Зачем первым |
|---|---|---|
| 0 | `indexnowkit/spec`: эта спецификация, mock-сервер, conformance YAML, README-шаблон, docs-сайт скелет | без этого адаптеры разъедутся |
| 1 | PHP: core, symfony-bundle (Doctrine), laravel | самая пустая ниша (Symfony) + самый большой спрос (Laravel); автор знает PHP |
| 2 | Python: core, django, sqlalchemy, fastapi, wagtail (обёртка над core вместо wagtail-indexnow) | Django ниша пустая, аудитория большая |
| 3 | JS: core, prisma, next, nuxt, typeorm, nestjs, drizzle, sveltekit, payload, strapi | много мелких адаптеров, ценность в покрытии |
| 4 | Rails, Spring Boot starter, EF Core, Go (gorm) | закрыть матрицу |
| 5 | Опционально: `google-indexing-api` адаптер (только JobPosting/BroadcastEvent, с явным `content_type`), sitemap-lastmod интеграции, API Platform, Drupal-модуль | по спросу |

## Definition of Done для пакета

- Conformance: все применимые сценарии из 03 зелёные в CI с mock-сервером.
- README по шаблону, RU-версия для PHP/Python/JS.
- Команда `check` реализована.
- Матрица версий фреймворка в CI (минимум 2 последних major/LTS).
- Static analysis на максимуме (phpstan level 9 / mypy strict / tsc strict).
- Опубликован в реестр, зарегистрирован минимум в двух каталогах из 90.
- Ссылка добавлена в таблицу семейства во всех остальных README.

## Метрики успеха (12 месяцев)

- Позиция 1–3 по запросу «<framework> indexnow» для 6 фреймворков.
- Суммарно ≥ 50k загрузок/мес по семейству.
- ≥ 1 внешний контрибьютор на адаптер.

## Решения, принятые 2026-09-03

- Бренд `indexnowkit`. Свободно: GitHub org, Packagist vendor, PyPI, RubyGems, домены
  indexnowkit.dev/.com/.io (нет DNS). npm-профиль не проверен (403 от бот-защиты): проверить
  при `npm org create`.
- GitHub org + монорепо на язык.
- Порядок: PHP (core → doctrine → symfony-bundle → laravel) → JS (core → next → prisma) →
  Python (core → django → sqlalchemy → fastapi). Остальное по спросу.
- Версии: PHP 8.2, Node 20, Python 3.11, Django 4.2+. Dispatch авто: queue при наличии
  Messenger/Laravel Queue, иначе sync после ответа.
- Документация EN основная, README.ru.md, MIT, VitePress на indexnowkit.dev.
- Старт: сразу PHP core + Symfony bundle; mock-сервер сначала как PHP-тестовый (встроенный
  сервер в тестах), выделение в Go-бинарник позже.
- WordPress вне scope подтверждено.

## Статус реализации (2026-09-03)

**0.2.0 — переработана, ещё не отмечена тегом.** Все три PHP-пакета переписаны вокруг модели правил
(`UrlRule`, повторяемый `#[IndexNow]`, `#[IndexNowDefaults]`, `#[IndexNowUrl]`, `ObjectChangeHandler`), фасад
переименован в `IndexNowKit`, у `Result` появилась причина (`Reason`), `TransactionStaging` переехал в core.
Ломающие изменения перечислены в `php/CHANGELOG.md`; публикации в Packagist пока не было.

- `php/packages/core`: 0.2.0 после аудита по семи линзам (API, протокол, безопасность, надёжность, тесты,
  документация, производительность) плюс переработка модели URL — см. `php/packages/core/CHANGELOG.md`.
  Conformance C01–C22 (C13 через `RetryingSubmitter`), phpstan 9, PHP 8.2–8.4, README EN/RU, `docs/`
  (конфигурация, справочник по атрибутам, повторы и очереди, эксплуатация, тестирование, гайд автора
  адаптера, BC), тестовые двойники `IndexNowKit\Testing` в публикуемом пакете, split-репо с собственным
  CI/SECURITY/CHANGELOG.
- `php/packages/doctrine`: 0.2.0. A01–A14 на sqlite плюс A15–A20 (несколько правил, удаление по правилу,
  `when`-геттер, удаление черновика, `via`, изменение коллекции), ORM 3 + DBAL 4 (DBAL 3 в CI-матрице),
  README EN/RU, собственный CHANGELOG.
- `php/packages/symfony-bundle`: 0.2.0. H01–H03, A01/A02/A04 функционально, Messenger (C13), полное дерево
  конфигурации с проверками на этапе компиляции, команды
  `indexnow:key:generate|check|submit|submit-entity|explain|sitemap`, Web Profiler панель с результатами,
  `docs/` (конфигурация, мультидомен, Messenger, HTTP-клиент, Doctrine, свои резолверы, тестирование,
  диагностика), Flex-рецепт в `recipe/`, split-workflow для Packagist
  (`.github/workflows/split.yml`, deploy-ключи в секретах SPLIT_SSH_KEY_*).
  `indexnowkit/doctrine` переведён из require в suggest.
- Опубликовано 2026-09-03: GitHub org `indexnowkit` (репо php, php-core, php-doctrine, php-symfony-bundle, spec), npm org
  `indexnowkit`, Packagist `indexnowkit/core|doctrine|symfony-bundle` 0.1.0. E2E-установка в чистый Symfony 7.4 skeleton
  с DoctrineBundle 3.3 проверена. Автообновление Packagist: GitHub-хуки установлены самим Packagist (OAuth-доступ приложения к org выдан, sync через packagist.org/trigger-github-sync/).
- `php/packages/laravel`: 0.2.0 (2026-09-04). Синхронный observer + `Connection::afterCommit()` (см. 13), `EloquentSubjectReader`
  через новый `SubjectReaderInterface` core, мост роутера с route model binding, `SubmitUrlsJob`, команды как в бандле,
  `config/indexnow.php`, README EN/RU + docs. Общий ORM-кит `OrmConformanceTestCase` (A01–A21) в core 0.2.2, Doctrine
  переведён на него.
- Не сделано: PR Flex-рецепта в recipes-contrib (отложено пользователем).

- `php/packages/yii2`: 0.1.0 (2026-09-04, спека 15). Компонент + behavior, verify-on-commit через core `Transaction\VerifyingStaging`
  (у Yii2 нет событий savepoint'ов), `php yii indexnow/*` над раннерами core, yii2-queue. Кит A01–A21 проходит. Yii3 — следующий (та же спека).
- Волна A спеки 16 — **выпущена 2026-09-05**: core 0.4.0, sitemap 0.1.0, doctrine 0.3.0, symfony-bundle 0.4.0, laravel 0.5.0, yii2 0.2.0 на Packagist (репо `php-sitemap`, split-CI зелёный, GitHub releases из changelog'ов): вынос sitemap в `indexnowkit/sitemap`, `Adapter\ConfigFactory`,
  статические фабрики, мелкие общие блоки; затем core 0.5.0 — `Adapter\Services`, `Hook\ObserverHelper`, `Retry\WorkerOutcome`,
  `Console\Definitions`.
- Волна B спеки 16 — **выпущена 2026-09-05**: core 0.5.0, sitemap 0.1.1, doctrine 0.3.1, symfony-bundle 0.5.0, laravel 0.6.0,
  yii2 0.3.0 на Packagist (additive в core): `Adapter\ServicesBuilder`/`Services` (Yii2 на них), `Hook\ObserverHelper`,
  `Retry\WorkerOutcome`, `Console\Definitions`, `Testing\KeyFileAssertions`/`CheckOutputAssertions`, coverage-floor в CI.
- Волна C спеки 16 — **выпущена 2026-09-05**: core 0.5.1, symfony-bundle 0.6.0, laravel 0.7.0, yii2 0.4.0:
  `indexnowkit/sitemap` снова `suggest` в адаптерах (`Check\StaticCheck`, `ConfigFactory(ignoreBlocks:)`, заглушка
  команды, sitemap-проводка за предикатом). Yii3 и Битрикс — следующие (спека 15, core 0.5 как база).
- Долги после волн A–C — **закрыты 2026-09-05**: symfony-bundle 0.6.1, yii2 0.5.0 (без релиза core/sitemap/doctrine/laravel):
  `split.yml` в три стадии (core → sitemap → адаптеры) с ожиданием dev-main на Packagist (`bin/packagist-wait-main`);
  help опций Yii из `Console\Definitions`; phpstan doctrine на флейворе dbal3 (`phpstan.dbal3.neon`, без baseline);
  `IndexNowKitLoader::load()` по блокам (`ContainerShapeTest`), `Yii2\Wiring`/`References` из компонента;
  `SubmitUrlsJob` yii2 перепушивает остаток с задержкой `Retry-After`/`retry.*`. Остаётся до 1.0:
  `ParamExtractor::registerReader()` (статическая регистрация, спека 16 §0).

- Спека 17, волна 0a + hotfix — **выполнена 2026-09-05**: core 0.6.0 (`check` красный на стейджинге с боевым ключом без
  `dry_run`, `Config::$dryRunExplicit`, дефект дебаунса при нескольких движках, `Engine::InternetArchive`/`Amazon`,
  `INDEXNOW_PREVIOUS_KEY`, 12 текстов ошибок, «Next:» в `check`), symfony-bundle 0.7.0 (`DebounceStoreCheck` +
  `CacheProbe`, узел `dry_run` без дефолта), laravel 0.8.0 (`env('INDEXNOW_DRY_RUN')` без каста, 12 | 13), yii2 0.6.0,
  doctrine 0.4.0, sitemap 0.2.0 (Psalm taint по расписанию); README-дефекты, «Why this over X», «≠ индексация»,
  `docs/bc.md` адаптеров; `bin/packagist-check`, Dependabot, `composer audit`, `roave/security-advisories`, шаблоны
  issue/PR, CoC, SECURITY SLA, бейджи. Уточнения реализации — спека 17 §11.
- Спека 17, волна 0b — **выполнена 2026-09-05** (без релизов; доки уедут патчами адаптеров): «Notes for AI assistants» ×6
  + `ReadmeAiNotesTest`, `AGENTS.md`, Boost-guideline, `.phpstorm.meta.php`, `context7.json`; quickstart-фикстуры
  `tests/Readme/Post.php` ×3 + `ReadmeQuickstartTest`; `bin/config-table` + «One concept, three keys»; docs-сайт
  (`bin/docs-collect`, MkDocs Material, `docs.yml`, lychee, `llms.txt`); prod checklist, SEO-тексты, антипаттерны,
  мониторинг; Yii2-паритет (configuration/testing/multi-domain/troubleshooting); RU ×5. Уточнения — спека 17 §12.
  Осталось пользователю: включить Pages, submit в Context7, внешний проход quickstart Yii2. Дальше: D (core 0.7.0).
- Спека 17 (2026-09-05, v2 после двух адверсальных ревью): путь к 1.0 — волна 0a+hotfix (core 0.6.0: стейджинг-проверка,
  дефект дебаунса, Engine ×2, тексты), 0b (доки, AI-разделы README, docs-сайт), D (core 0.7.0: пакеты testing/console,
  OptionalPackage), E (core 0.8.0: check --json/--strict, ротация, счётчик 403, SubmissionStoreInterface, канонизация,
  Condition), F (verify, history), затем 0.9 без breaking → 1.0. Yii3/Битрикс после.

## Открытые решения

1. Trademark «IndexNow» у Microsoft: допустимо ли в имени бренда. Проверить до первой публикации.
2. Wagtail: PR в `wagtail-indexnow` или свой пакет. Дефолт: PR + обёртка через 30 дней.
3. Google Indexing API адаптер (JobPosting/BroadcastEvent): только по спросу.
