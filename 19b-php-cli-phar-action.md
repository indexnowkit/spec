# 19b. Волна N (фаза B спеки 18): `indexnow` — CLI без фреймворка, PHAR, Docker-образ, GitHub Action

Статус: **выпущен 2026-09-08/09** (решения §9 по рекомендациям): core 0.13.1, sitemap 0.9.0, cli 0.1.0 на Packagist; релиз
`php-cli` 0.1.0 с `indexnow.phar`; образ `ghcr.io/indexnowkit/indexnow` (`0.1.0`, `0.1`, `latest`, `0.1.0-action`) public;
действие `indexnowkit/indexnow-action` в Marketplace как «IndexNow submit (indexnowkit)» на `v1.0.1` (`v1`): Marketplace
режет description `action.yml` на 125 символах — `action@1.0.0` не прошёл, `action@1.0.1` укоротил его и ничего больше;
тег, запушенный сплитом через deploy-key, не запускает `release.yml` сплита (перепуш своим ключом).
Отклонения от §3: (1) `.env` **парсится** (`Dotenv::parse()`), не загружается в процесс — реальное окружение накладывается
явно, `$_ENV`/`putenv` не трогаются, `variables_order` неважен (§7.7 снят); (2) entrypoint действия — **PHP** (`docker/indexnow-action`),
не shell: GitHub передаёт входы как `INPUT_BASE-URL` с дефисом (проверено по docs.github.com), sh такое не читает; `dry-run`
действия — флаг команды, не `INDEXNOW_DRY_RUN` (иначе `check` в prod считает dry run ошибкой); (3) образ в **двух target**:
`cli` с `USER indexnow` (uid 1000) и `<ver>-action` без `USER` — Docker-действие обязано работать от root
(docs.github.com «Dockerfile support», `USER`), `action.yml` пинит `-action`; (4) в `check` нет строки `verify.dispatch`
(её текст советует очередь, которой у процесса нет), `--sample-class` отвечает текстом пакета verify; `status` описывает
стор как `state (SqliteCache)` через `HistoryServices::describeStore()`; (5) `SitemapRunner` запоминает батчи и **без**
`--new-only` (полный прогон, затем `--new-only` в cron — каждое изменение один раз); (6) `history.store` выключается
`INDEXNOW_HISTORY_STORE=` / `none` / `null` / `off`; (7) `bin/config-table` без колонки «CLI env» — правило одно
(`INDEXNOW_<BLOCK>_<KEY>`), таблица в `cli/docs/configuration.md`; (8) `CommandDefinition`-стаб `ConfigurationErrorCommand`:
`help <cmd>` работает без ключа, сама команда печатает ошибку конфигурации; (9) `require-dev` четырёх адаптеров —
`indexnowkit/sitemap ^0.8 || ^0.9` (path-репозиторий 0.9.x-dev иначе не резолвится в монорепо-CI; релиза адаптеров не нужно);
(10) попутно в core 0.13.1: `Config::unknownOptions()` спускается во вложенные блоки — `history.pdo` считался неизвестным
во всех адаптерах; (11) coverage-floor cli записан локально (93.75), перезаписать числом CI после пуша (урок волны M);
mutation для cli нет (решение 8). Инфраструктура `gh` создана: `indexnowkit/php-cli`, `indexnowkit/indexnow-action`
(public, issues/wiki/projects off, topics), deploy-keys «split», секреты `SPLIT_SSH_KEY_CLI`, `SPLIT_SSH_KEY_ACTION`.

## 0. Цель и границы

Один бинарник `indexnow`, который работает на любом хосте с PHP 8.2 и в любом CI без фреймворка: cron на Битриксе,
WordPress, MODX, OpenCart, Joomla, деплой статики (Hugo, Astro, Jekyll), «просто отправить десять URL руками». Три упаковки
одного и того же: Composer-пакет `indexnowkit/cli` (`vendor/bin/indexnow`, `composer global require`), `indexnow.phar`
(релиз-ассет), Docker-образ `ghcr.io/indexnowkit/indexnow`. Поверх образа — GitHub Action `indexnowkit/indexnow-action`
(Marketplace): «отправить изменившиеся URL на деплое».

Что это даёт семейству: (1) ширина без адаптеров — любая CMS с sitemap покрыта в этой волне, Битрикс в том числе (он
генерирует `sitemap.xml` и индекс по инфоблокам штатно, модуль «Поисковая оптимизация» с версии 14 —
dev.1c-bitrix.ru/community/blogs/product_features/seo-and-sitemapxml-in-version-14-1cbitrix-site-management.php); (2) канал
обнаружения вне PHP — Actions Marketplace и GHCR; (3) сигнал «что где и как» — история и `status` на голом хосте
(файл состояния), которых у конкурентов нет.

Границы: без модуля Битрикса (следующая волна), без `explain`/`submit-<subject>` (нужен `SubjectLoaderInterface`, у голого
приложения его нет — спека 18 §8), без phive/GPG-подписи (нужен ключ пользователя; §8), без WordPress-плагина (спека 44).

## 1. Что есть сейчас (факты, проверены 2026-09-08)

### 1.1. Команды уже не знают фреймворка (волны L, M)

- Конструкторы классов команд берут только раннеры и три «переменные адаптера»: `Console\Command\CheckCommand(CheckRunner,
  ConfigSourceInterface, ?SampleOptions)` (`console/src/Command/CheckCommand.php:31`), `ConfigCommand(ConfigRunner,
  ConfigSourceInterface)` (`:25`), `SubmitCommand(SubmitRunner)` (`:22`), `KeyGenerateCommand(KeyGenerateRunner, string
  $envFileName = '.env', ?string $envFile = null)` (`:31`), `Sitemap\Console\SitemapCommand(SitemapRunner, string
  $sitemapUrlOption = 'sitemap.url')` (`sitemap/src/Console/SitemapCommand.php:27`), `History\Console\HistoryCommand(HistoryRunner)`
  (`:27`), `StatusCommand(StatusRunner)` (`:24`). Голое приложение регистрирует эти классы и не пишет своих — спека 18 §3.1,
  `core/docs/adapters.md` §14.
- `Console\ConfigSourceInterface` — три метода: `raw()`, `build()`, `packages()` (`console/src/ConfigSourceInterface.php:16-41`).
  Реализации: бандл (над деревом конфигурации), Yii3 (`yii3/src/Console/ConfigSource.php:17-38`, над фасадом), Laravel
  (`laravel/src/Console/ConfigSource.php`). Спека 18 §8 предсказала единственную точку фазы B — `EnvConfigSource`; так и есть.
- `Console\Definitions` (`console/src/Definitions.php`) и `Sitemap\Console\Definitions` (`sitemap/src/Console/Definitions.php:22-36`)
  объявляют аргументы и опции один раз; `History\Console\Definitions` — для `history`/`status`. Тексты помощи не дублировать.
- `Console\Vocabulary` (`console/src/Vocabulary.php:20-33`): `cli = 'indexnow'`, `check = 'indexnow:check'`, `submit`,
  `explain`, `configLocation`, `keyFileServedBy` — все тексты раннеров берут слова отсюда; у CLI имена команд без префикса
  `indexnow:` (префикс — сам бинарник).
- Раннеры: `CheckRunner::run($io, callable $validateConfig, bool $live, $host, $probeUrl, bool $json, …)`
  (`console/src/CheckRunner.php:45`), `ConfigRunner::run($io, callable $buildConfig, array $raw, bool $json, array $packages)`
  (`:35`), `SubmitRunner::run($io, array $urls, bool $force, bool $dryRun, bool $json)` (`:28`), `KeyGenerateRunner::run($io,
  int $length, bool $hex, ?string $envFile, bool $force, bool $noPrevious, …)` (`:37`) — пишет `.env` через `createPrivate()`
  (`:39`), `StatusRunner::run($io, bool $json)` (`history/src/Console/StatusRunner.php:55`), `SitemapRunner::run($io,
  SitemapOptions)` (`sitemap/src/Console/SitemapRunner.php:65`).

### 1.2. Конфигурация из окружения покрывает только ядро

- `Config::fromEnv(?array $env, string $prefix = 'INDEXNOW_')` (`core/src/Config.php:394`) → `ConfigParser::fromEnv()`
  (`core/src/Config/ConfigParser.php:130-165`): 30 переменных ядра (`INDEXNOW_KEY`, `INDEXNOW_HOSTS "host=key,…"`,
  `INDEXNOW_BASE_URL`, `INDEXNOW_ENGINES`, `INDEXNOW_DEBOUNCE_STORE`, `INDEXNOW_HTTP_CLIENT`, … — список в докблоке
  `Config.php:380-388`), собирает **вложенный массив** и зовёт `fromArray()`. Массив наружу не отдаёт: только `Config`.
- Блоки пакетов из окружения не читаются ничем: `grep -rn "function fromEnv" packages/*/src` — только ядро. Ключи блоков
  объявлены списками: `SitemapConfig::OPTIONS` (9 ключей, `sitemap/src/SitemapConfig.php:21-24`), `VerifyConfig::OPTIONS`
  (11, `verify/src/VerifyConfig.php:21-24`), `HistoryConfig::OPTIONS` (`history/src/HistoryConfig.php:18`, в том числе
  вложенные `history.pdo.dsn`, `history.pdo.table`). Каждый `*Config::fromArray()` коэрсит строки (`"3"`, `"true"`) сам
  (`SitemapConfig::fromArray()` `:84-107`) — переменные окружения можно отдавать строками.
- `Config::toArray()` (`Config.php:432+`) отдаёт **все** опции с разрешёнными значениями — для слияния «env поверх файла»
  не годится (дефолты перекроют файл). Нужен массив «только то, что задано» — §3.3.

### 1.3. Граф без фреймворка: adapter kit

- `IndexNowKit::create(Config, …16 именованных)` (`core/src/IndexNowKit.php:106-140`): транспорт по умолчанию
  `TransportFactory::lazy($config)` (`:128`) → `Psr18Transport::discover()` через `php-http/discovery`
  (`core/src/Http/TransportFactory.php:34-52`). Ядро требует `php-http/discovery ^1.20`, но его composer-плагин у всех
  пакетов выключен (`allow-plugins.php-http/discovery: false`), а реализации только в `suggest` (`core/composer.json:56-59`:
  `symfony/http-client`, `guzzlehttp/guzzle`, `nyholm/psr7`). Голому CLI клиент надо **требовать явно** и отдавать явно
  (`lazy(…, requestFactory:, streamFactory:)` — волна M, §4.8 спеки 19; клиент — через `ServicesBuilder::transport()`).
- `Adapter\ServicesBuilder(Config, LoggerInterface)` (`core/src/Adapter/ServicesBuilder.php:64`) с сеттерами узлов
  (`transport`, `debounceStore`, `failureCache`, `submissionStore`, `clock`, `checks`, …, `:70-246`) и `Adapter\Services`
  (`core/src/Adapter/Services.php`): `checker()` (`:309` — `new Checker($config, $keys, $transport, $checks)`),
  `submitterFactory()` (`:323`), `kit()` (`:287`), `keyFileResponder()` (`:304`), `forbiddenCounter()` (`:173`). Yii3
  `Wiring` — образец второго слоя (спека 17 §16.4).
- Опциональные пакеты дают статические фабрики поверх графа: `Sitemap\Adapter\SitemapServices::reader/readerFor/runner/command/
  spoolCheck` (`sitemap/src/Adapter/SitemapServices.php:63-93`), `History\Adapter\HistoryServices::pdoFromDsn/pdoStore/storeFor/
  historyRunner/statusRunner/describeStore/checksFor` (`history/src/Adapter/HistoryServices.php:76-218`),
  `Verify\Adapter\VerifyServices::transport/submitterFactory/checksFor/sampleCheck` (`verify/src/Adapter/VerifyServices.php:84-176`).
  `OptionalPackage::ownedOptions()/ignoredBlocks()` (`core/src/Adapter/OptionalPackage.php:105-118`) — для списка допустимых ключей.
- Дебаунс: `DebounceStoreFactory::fromConfig($config, ?Closure $cacheLocator, $default = MEMORY, ?ClockInterface)`
  (`core/src/Debounce/DebounceStoreFactory.php:46`): `memory` (на процесс), `none`, иначе id PSR-16 через локатор. **Cron-запуск —
  новый процесс: без PSR-16 у CLI нет дебаунса между запусками**, каждый прогон `sitemap` без `--changed-since` шлёт всё заново.
  Файловой или sqlite-реализации PSR-16 в семействе нет (`core/src/Debounce/`: Memory, Null, Psr16).
- История: `History\Pdo\PdoSubmissionStore(PDO)` с `createTable()`; `history.pdo.dsn` принимает `sqlite:var/indexnow.sqlite`
  (`history/docs/configuration.md:14`, `HistoryServices::pdoFromDsn()` `:76`). Один sqlite-файл может нести и историю, и кэш.

### 1.4. Sitemap: только `lastmod`

- `SitemapReader::read($url, ?DateTimeImmutable $since, ?bool $allowForeign)` фильтрует по `<lastmod>`; записи **без**
  `lastmod` при заданном `$since` пропускаются (`sitemap/README.md:56-58`). `SitemapRunner::changedSince()`
  (`SitemapRunner.php:197-207`) — относительные окна на часах графа. Полный прогон больше батча — предупреждение «engines see
  every page as changed» (`:170-173`). Никакого «что изменилось с прошлого запуска» нет: у адаптеров cron сам выбирает окно.
- Конкуренты упираются в то же: `bojieyang/indexnow-action` (Node, v3 на Node 24; `since` + `since-unit`, `limit` 100 по умолчанию,
  `lastmod-required`, один endpoint за прогон, sitemap/index/RSS/Atom), `jakob-bagterp/index-now-submit-sitemap-urls-action`
  (Python, composite; `sitemap_days_ago`, regex-фильтр, один endpoint, 5★). Ни у одного: проверка ключ-файла перед отправкой,
  все движки за прогон, pre-flight, состояние между прогонами, история.

### 1.5. Ключ-файл на голом хосте

`core/README.md:94-105`: «`file_put_contents("public/$key.txt", $key)` или ответьте сами через `KeyFileResponder`».
Команды, которая пишет файл, нет. `KeyProviderInterface::managedHosts()/keyFor()/keyLocationFor()`
(`core/src/Key/KeyProviderInterface.php:19-40`) дают всё, чтобы записать `<key>.txt` для каждого хоста в document root.
`KeyGenerateRunner` печатает `keyFileServedBy` из словаря — у CLI это будет «файл, который пишет `key:file`».

### 1.6. Инфраструктура монорепо

- Новый пакет: репо `indexnowkit/php-<name>`, deploy-key, секрет `SPLIT_SSH_KEY_<NAME>`, строка в `split.yml`, Packagist
  после первого сплита (`php/README.md:102-104`). Стадии `split.yml`: core → testing, console → sitemap, verify, history →
  адаптеры; `packagist-wait-main` перед пушем main. `bin/link.php:41-45` требует `extra.branch-alias.dev-main`.
- Сплит-репозитории несут **собственный** `.github/workflows/ci.yml` внутри каталога пакета
  (`packages/core/.github/workflows/ci.yml`: PHP 8.2–8.5 highest + 8.2 lowest); в монорепо вложенные workflow не запускаются.
  Значит `packages/cli/.github/workflows/release.yml` будет работать в `php-cli` с его `GITHUB_TOKEN` — PHAR в релиз и образ
  в GHCR без секретов монорепо.
- `bin/ci` (`php/bin/ci`): link → `ci:install:<flavour>` → phpunit → phpstan src (level 9) → phpstan tests (`phpstan.tests.neon`).
  Матрица `ci.yml:20-46`: 11 пакетов × 8.2–8.5 + lowest на 8.2. `bin/docs-collect:21-22` — списки `PACKAGES`/`TITLES`,
  `GETTING_STARTED` (`:26`). `testing/tests/Unit/ConformanceIdsTest.php:80` — список библиотек, которые не адаптеры: новый
  пакет с `tests/` и `require indexnowkit/core` иначе считается адаптером и должен нести H01–H06.
- Образ `php:8.3-cli` (`docker/php/Dockerfile`) — расширения из коробки: curl, mbstring, openssl, pdo_sqlite, Phar, sodium,
  xmlreader, zlib; **intl нет** (Punycode — чистый PHP-фолбэк, `core/composer.json` suggest). Инструменты монорепо — `tools/psalm`,
  `tools/infection` со своими lock; `tools/box` — по образцу.
- `bin/tag` (`php/bin/tag`) пушит поддерево и тег `<pkg>@<ver>`; `bin/release-notes --create` делает `gh release create` и
  **падает, если релиз уже есть**; `bin/packagist-wait` ждёт p2 (CDN до 15 минут — спека 17 §16.6).

### 1.7. Внешние факты

- **Box** (`humbug/box`) 4.7.0, 2026-03-18, `php ^8.2` (packagist.org/packages/humbug/box). `box.json`: `main` (из `bin`
  composer.json), `output`, `compactors` (`KevinGH\Box\Compactor\Php`, `Json`), `compression`, `git-version`/`git-commit-short`
  плейсхолдеры (`@git-version@`), `dump-autoload` (classmap-authoritative), `check-requirements` — проверка PHP и расширений
  **из composer.lock** при запуске PHAR (`ext-pdo_sqlite`, `ext-xmlreader`, `ext-zlib`), полифилы учитываются; `exclude-dev-files`
  (box-project.github.io/box/configuration, /requirement-checker).
- **GitHub Marketplace**: публичный репозиторий, **один `action.yml` в корне**, уникальное `name` (не совпадает с существующим
  действием, пользователем/организацией, категорией; «IndexNow Action» уже занято bojieyang), `branding` (icon, color), 2FA и
  Developer Agreement при публикации; action в подкаталоге не листится (docs.github.com/…/publish-in-github-marketplace).
  Docker-действие: `runs: { using: docker, image: 'docker://ghcr.io/…:tag' }` — готовый образ, без сборки на раннере;
  `args`, `env`, `pre-entrypoint`; только Linux-раннеры (docs.github.com/…/metadata-syntax).
- **Протокол** (indexnow.org/documentation): до 10 000 URL на POST; ключ 8–128 символов `[A-Za-z0-9-]`; ключ-файл в корне или в
  подкаталоге (тогда только URL под ним); 200/202/400/403/422/429. Уже в спеке 01 и `Config`.
- **phive** (phar.io): `.phar` + `.phar.asc` (GPG detached) в релизе GitHub — отложено (§8).

## 2. Принципы

1. **CLI — composition root, не библиотека.** Тексты, опции, раннеры — в `console`/`sitemap`/`history`/`verify`/`core`; в
   `indexnowkit/cli` живут только проводка, чтение окружения, файл состояния и две команды, которых нет ни у кого (`key:file`,
   `--new-only` — см. §3.4, §3.5: логика — в пакетах, CLI — реализация хранилища).
2. **Три упаковки одного бинарника.** PHAR, образ и Action не расходятся по поведению: Action — это `indexnow sitemap --json` в
   образе, ничего сверх.
3. **Детерминированный транспорт.** Без `php-http/discovery` в рантайме: `symfony/http-client` + `nyholm/psr7` в `require`,
   переданы явно. PHAR не должен зависеть от того, что нашлось в classmap.
4. **Одно место состояния.** `.indexnow/state.sqlite` в рабочем каталоге (переопределяется): дебаунс, счётчик 403, история,
   «виденные» URL sitemap. Всё, что делает cron-прогоны идемпотентными, — в одном файле, который легко положить в `actions/cache`.
5. **Окружение — первый источник, файл — второй.** `INDEXNOW_*` для всего (ядро и блоки пакетов по одному правилу),
   `.env` рядом, JSON-файл для сложного (карта хостов). Приоритет: опции команды > реальное окружение > `.env` > `--config` > умолчания.
6. Правила семейства: тиры `bc.md`, конструкторы растут именованными аргументами, research-first, тексты «факт — что можно —
   как починить», ключи маскируются.

## 3. Дизайн

### 3.1. Пакет `indexnowkit/cli` 0.1.0

`composer.json`: `"bin": ["bin/indexnow"]`; `require`: `php ^8.2`, `ext-pdo_sqlite`, `ext-xmlreader`, `indexnowkit/core ^0.14`,
`indexnowkit/console ^0.5`, `indexnowkit/sitemap ^0.9`, `indexnowkit/verify ^0.4`, `indexnowkit/history ^0.4`, `symfony/console
^6.4||^7.0||^8.0`, `symfony/dotenv ^6.4||^7.0||^8.0`, `symfony/http-client ^6.4||^7.0||^8.0`, `nyholm/psr7 ^1.8`, `psr/simple-cache`,
`psr/log`; `suggest`: `ext-intl`, `ext-zlib`; `require-dev`: `indexnowkit/testing`, phpstan/phpunit как у console;
`extra.branch-alias.dev-main: 0.1.x-dev`. Namespace `IndexNowKit\Cli\`. Тир `bc.md` пакета: всё Call, кроме `State\*` — internal
до 1.0 (публичный контракт CLI — команды и переменные, не классы).

Классы (composition root, каждый ≤ 200 строк):

- `Cli\Application extends Symfony\Component\Console\Application` — имя `indexnow`, версия: `Composer\InstalledVersions::getPrettyVersion('indexnowkit/cli')`
  в Composer-установке, `@git-version@` (box) в PHAR — `Cli\Version::current()` выбирает. Глобальные опции: `--env-file=<path>`
  (по умолчанию `.env` в cwd, если есть), `--no-env-file`, `--config=<file.json>`, `--state=<path>`, `--working-dir`? — нет (cwd
  достаточно). Регистрирует восемь команд лениво (`LazyCommand`/`CommandLoader`, как бандл): `check`, `config`, `submit`,
  `key:generate`, `key:file`, `sitemap`, `history`, `status`. Стабы `*NotInstalledCommand` не нужны — пакеты в `require`.
- `Cli\Env\Dotenv` — обёртка `Symfony\Component\Dotenv\Dotenv::load()` (реальное окружение выигрывает у файла; отсутствующий
  файл при явном `--env-file` — `ConfigurationException`, при неявном — тишина). `usePutenv(false)`: `Config::fromEnv()` читает
  `getenv() + $_SERVER + $_ENV` (`Config.php:396`) — `$_ENV` достаточно.
- `Cli\Config\EnvConfigSource implements ConfigSourceInterface` — `raw()`: слияние JSON-файла (`--config`/`INDEXNOW_CONFIG`) и
  окружения по правилу §3.3; `build()`: `Config::fromArray(raw без блоков пакетов)` через `Adapter\ConfigFactory` (та же строгая
  сборка, что у адаптеров, с `unknownOptions()` по `OptionalPackage::ownedOptions()`); `packages()`: `sitemap`, `verify`, `history`
  через `*Config::fromArray()->toArray()`.
- `Cli\Wiring` — по образцу `Yii3\Wiring`: `ServicesBuilder` над `Config`, транспорт `Psr18Transport::discover(new
  Symfony\Component\HttpClient\Psr18Client(...), $config->httpTimeout, requestFactory: new Nyholm\Psr7\Factory\Psr17Factory(),
  streamFactory: …)`, `http.client` → `ConfigurationException` «this CLI has no container to resolve an http.client id; unset it»;
  дебаунс `DebounceStoreFactory::fromConfig($config, $cacheLocator, default: 'state')` где локатор знает один id — `state` (§3.2);
  `failureCache` — тот же кэш; `submissionStore` — история §3.2; часы — системные; `checks`: `SitemapSpoolCheck`,
  `VerifyServices::checksFor()` (с `sampleCheck` над `SampleOptions` без сэмплера классов — `--sample-class` отвечает «no classes
  in a CLI without an ORM: give URLs with --sample»), `HistoryServices::checksFor()`, `DebounceStoreCheck` над кэшем состояния,
  `StaticCheck` «state: <path> (writable|read-only)». Логгер — `Psr\Log` в stderr (`Symfony\Component\Console\Logger\ConsoleLogger`),
  уровень по `-v`.
- `Cli\State\State` — путь (`--state` > `INDEXNOW_STATE` > `./.indexnow/state.sqlite`; значение `memory` = `sqlite::memory:` —
  для read-only контейнеров и `--dry-run`), открывает PDO, `PRAGMA journal_mode=WAL`, создаёт таблицы (`indexnow_cache`,
  `indexnow_submissions` через `PdoSubmissionStore::createTable()`, `indexnow_sitemap_seen`), отдаёт `SqliteCache`,
  `PdoSubmissionStore`, `SqliteSeenStore`. Создание каталога — `mkdir(0700)`; ошибка записи — одна `ConfigurationException`
  с путём и подсказкой `--state memory`.
- `Cli\State\SqliteCache implements Psr\SimpleCache\CacheInterface` — ~120 строк: `(key TEXT PK, value BLOB, expires_at INTEGER NULL)`,
  TTL через `ClockInterface`, ключи по PSR-16 (`InvalidArgumentException` на `{}()/\@:`), `getMultiple`/`setMultiple`/`deleteMultiple`,
  ленивая чистка просроченных при `set`. Решение 4: своя реализация, а не `symfony/cache` (десять пакетов ради одной таблицы;
  файл состояния и так sqlite).
- `Cli\State\SqliteSeenStore implements Sitemap\SeenStoreInterface` (§3.5).
- `Cli\Command\KeyFileCommand` — `key:file <docroot> [--host=<h>]* [--dry-run]`: для каждого `managedHosts()` пишет
  `<docroot>/<key>.txt` (или путь из `keyLocationFor()`, если он под этим хостом) телом `bodyForKey()`; печатает таблицу
  host → файл → «written | unchanged | would write»; предупреждение, если docroot не существует; после — «Verify with:
  indexnow check --live». Тир Call. Определение опций — `Cli\Definitions::keyFile()` по образцу `Console\Definitions` (единственная
  новая команда семейства; если её захотят адаптеры — переедет в `console` вместе с определением, конструкторы не поменяются).
- `Cli\Vocabulary` — не класс, а вызов: `new Vocabulary(subject: 'url', subjects: 'urls', cli: 'indexnow', submitSubjects: '',
  configLocation: 'the INDEXNOW_* variables (.env in the working directory) or the --config file', keyFileServedBy: 'by the
  <key>.txt file that "indexnow key:file <docroot>" writes', check: 'check', submit: 'submit', explain: '')`.

`bin/indexnow`: `#!/usr/bin/env php`, ищет автозагрузчик (`vendor/autoload.php` на трёх уровнях + `__DIR__/../vendor`), `exit((new
Application())->run())`.

### 3.2. Умолчания состояния (решение 3)

`debounce.store` по умолчанию `state` (не `memory`): второй cron-прогон не шлёт то, что послал первый десять минут назад — это
«окно дебаунса» протокола, и на голом хосте его больше некому держать. `history.store` по умолчанию `pdo` над файлом состояния
(`history.pdo.dsn` не задан → `sqlite:<state>`): `indexnow history` и `status` работают из коробки — это и есть «понимание что
где и как» на хосте без фреймворка. Отключение — `INDEXNOW_HISTORY_STORE=` (пусто) или `null`; `--state memory` делает всё
одноразовым. `check` печатает `debounce.store: state (.indexnow/state.sqlite)` через `HistoryServices::describeStore()`.

### 3.3. Окружение и файл: правило одно на все блоки (решение 2)

- Ядро: как есть — `Config::fromEnv()`. **Аддитивно в core 0.14.0**: `Config::arrayFromEnv(?array $env = null, string $prefix =
  'INDEXNOW_'): array` — тот же разбор, что `ConfigParser::fromEnv()` (`:130-165`), но отдаёт вложенный массив «только заданное»;
  `fromEnv()` становится `fromArray(arrayFromEnv())`. Тир Call, тест: `fromArray(arrayFromEnv($env))` равен `fromEnv($env)`.
- Блоки пакетов: `INDEXNOW_<BLOCK>_<KEY_PATH>` → `block.key.path` по спискам `SitemapConfig::OPTIONS`, `VerifyConfig::OPTIONS`,
  `HistoryConfig::OPTIONS` (`sitemap.max_depth` → `INDEXNOW_SITEMAP_MAX_DEPTH`, `history.pdo.dsn` → `INDEXNOW_HISTORY_PDO_DSN`).
  Правило общее, реализация — `Cli\Config\EnvBlocks::read(array $env, array $options, string $prefix): array` (~40 строк):
  имя переменной = префикс + путь в верхнем регистре с `_`; значение — строка как есть (коэрсия в `*Config::fromArray()`).
  Неизвестные `INDEXNOW_*` — предупреждение `check` (`config.unknown`, как у адаптеров через `unknownOptions()`), не ошибка.
- JSON-файл (`--config`, `INDEXNOW_CONFIG`): та же вложенная форма, что у `Config::fromArray()` плюс блоки `sitemap`/`verify`/
  `history` — форма адаптеров без фреймворка. Только JSON (PHAR-безопасно, без `include`). Слияние: `array_replace_recursive(json,
  env)`; `hosts` из env заменяет карту целиком (как `INDEXNOW_HOSTS` сейчас).
- `config --json` показывает источник каждого верхнего ключа? — нет (YAGNI); показывает `adapter: {config_file, env_file, state}`.

### 3.4. Команда `key:file`

Единственная новая команда. Мотив: на голом хосте это первый шаг после `key:generate`, и сейчас он в README как две строки PHP.
Пишет только в переданный docroot (аргумент оператора — тот же класс потока, что `--write-env`; в `psalm.xml` подавление с
причиной, как у `KeyGenerateRunner`).

### 3.5. `sitemap --new-only` и `Sitemap\SeenStoreInterface` (sitemap 0.9.0, решение 5)

Мотив: `lastmod` есть не у всех (Битрикс ставит; многие генераторы — нет; bojieyang завёл флаг `lastmod-required`), а «окно в
один день» на деплое статики промахивается в обе стороны. Состояние решает: отправлять то, чего в прошлый раз не было или что
изменилось.

- `Sitemap\SeenStoreInterface` (тир Implement, растёт только новым интерфейсом): `unseen(iterable<SitemapEntry> $entries):
  iterable<SitemapEntry>` — те, чей отпечаток (`url` + `lastmod` ISO или пусто) отсутствует или отличается; `remember(iterable<
  SitemapEntry> $entries): void`; `forget(): void` (для `--force`? нет — `--force` про дебаунс; `forget()` не нужен, убрать).
- `SitemapRunner::__construct(…, ?SeenStoreInterface $seen = null)` — десятый, именованный. `SitemapOptions::$newOnly`.
  `Sitemap\Console\Definitions::sitemap()`: `OptionDefinition::flag('new-only', 'Submit only the URLs that are new or changed since
  the last run of this command (an adapter without a store of seen URLs says so)')`. Раннер: `--new-only` без стора → `INVALID` с
  текстом «--new-only needs a store of seen URLs; this application has none (the CLI keeps one in its state file)»; со стором —
  `unseen()` после `onManagedHosts()` и до батчей; `remember()` только для батчей, чьи результаты не failed (`ResultSummary`);
  `--dry-run` не запоминает; `--changed-since` и `--new-only` складываются. Сообщение `foundLine` дополняется «, N new or changed».
- Адаптеры в этой волне стор не получают (опция принята, ответ — текст выше); заметка в `sitemap/docs/adapters.md`: PSR-16
  (`Psr16SeenStore` — нет, отпечатки должны переживать TTL; PDO — таблица) — следующая волна, по спросу.
- CLI: `SqliteSeenStore` — `(url TEXT PK, fingerprint TEXT, seen_at INTEGER)`, `unseen()` — генератор с `SELECT fingerprint WHERE
  url = ?` по мере потока (без загрузки списка), `remember()` — `INSERT OR REPLACE` в транзакции по батчу.

### 3.6. PHAR

- `packages/cli/box.json`: `main: bin/indexnow`, `output: indexnow.phar`, `compactors: [Php, Json]`, `compression: GZ`?
  (требует `ext-zlib` у пользователя — есть везде; да), `git-version` → `Cli\Version` через `@git-version@`, `check-requirements:
  true` (из `composer.lock` сборки: php ^8.2, pdo_sqlite, xmlreader, zlib), `exclude-dev-files: true`, `directories: [src, bin,
  vendor]` + `files: [composer.json]`; `blacklist`: `tests`, `docs`, `README*`.
- `tools/box/composer.json` (`humbug/box ^4.7`), `bin/phar [cli]` (Docker): `composer install --no-dev --classmap-authoritative` в
  копии пакета (не трогать `vendor` dev-сборки — работать во временном каталоге сессии/`var/phar`), затем `box compile`; проверка:
  `php indexnow.phar --version`, `php indexnow.phar list`. `composer.lock` пакета не коммитится (как у всех) — box берёт
  требования из lock временной установки.
- Где строится: **в сплит-репозитории** `php-cli`, `packages/cli/.github/workflows/release.yml` на тег `[0-9]*.[0-9]*.[0-9]*`:
  setup-php 8.3 → `composer install --no-dev` → `composer install -d tools/box`? — `tools/` в сплит не попадает → `box` ставится
  через `phive`/`composer global require humbug/box:^4.7` в джобе → `box compile` → `sha256sum` → `gh release upload <tag>
  indexnow.phar indexnow.phar.sha256 --clobber` (релиз создаёт `bin/release-notes --create` из монорепо; джоба ждёт его до 10
  минут, иначе создаёт сама `gh release create --notes 'see CHANGELOG'` — тогда `bin/release-notes` должен уметь `edit`:
  **правка `bin/release-notes`: если релиз есть — `gh release edit --notes-file -`**). Второй job — Docker (§3.7).
- Монорепо CI: джоба `cli / phar` (`ci.yml`): собирает PHAR так же (`bin/phar` логика в shell-шагах), гоняет `php indexnow.phar
  check --json` с `INDEXNOW_KEY`, `INDEXNOW_BASE_URL=http://127.0.0.1:8089`, `INDEXNOW_ENGINES=…` против mock-сервера `php -S
  127.0.0.1:8089 packages/testing/resources/mock-server/router.php` (`php/README.md:76-80`), и `submit http://127.0.0.1:8089/a`
  → `--json` со статусом `ok`. Это единственный тест PHAR-специфики (автозагрузка, `php-http/discovery` не задействован,
  `Phar::running()` пути состояния).

### 3.7. Docker-образ `ghcr.io/indexnowkit/indexnow`

- `packages/cli/Dockerfile`, multi-stage: `composer:2` + `php:8.3-cli-alpine` для сборки PHAR (или COPY готового `indexnow.phar`
  из джобы — проще: сборочный stage = те же шаги, что §3.6), финальный `php:8.3-cli-alpine`: `COPY indexnow.phar
  /usr/local/bin/indexnow`, `chmod +x`, пользователь `indexnow` (uid 1000), `WORKDIR /work` (том с `.indexnow/` и sitemap-файлами),
  `ENTRYPOINT ["indexnow"]`, `CMD ["list"]`. `docker-php-ext-install intl` — нет (icu-dev +30 МБ; Punycode-фолбэк ядра достаточен,
  документируем `ext-intl` как отличие); `pdo_sqlite`, `xmlreader`, `zlib`, `curl` — в базовом образе.
- Публикация: тот же `release.yml` сплита: `docker/login-action@v3` (GHCR, `GITHUB_TOKEN`, `permissions: packages: write`),
  `docker/metadata-action@v5` (теги `<ver>`, `<major>.<minor>`, `latest`), `docker/build-push-action@v6` (linux/amd64 + arm64
  через buildx/QEMU), `actions/attest-build-provenance` — опционально. Первый пуш создаёт пакет GHCR **приватным** — публичным
  его делает пользователь в настройках пакета организации (шаг «кроме создания репы»).
- Монорепо CI: джоба `cli / docker` — `docker build` без пуша, smoke: `docker run --rm image --version`, `docker run --rm -e
  INDEXNOW_KEY=… -e INDEXNOW_STATE=memory image check --json` (без `--live`).

### 3.8. GitHub Action `indexnowkit/indexnow-action`

- Источник — `packages/cli/action/` в монорепо, зеркалится сплитом в репо `indexnowkit/indexnow-action` (Marketplace: `action.yml`
  в корне репозитория — значит корень зеркала = этот каталог): `action.yml`, `entrypoint.sh`, `README.md`, `LICENSE`.
  `split.yml`: псевдо-пакет `action` в стадии после адаптеров (`prefix: packages/cli/action`, секрет `SPLIT_SSH_KEY_ACTION`), тег
  `action@1.2.3` → `v1.2.3` **и** движущийся `v1` (Marketplace-конвенция `uses: indexnowkit/indexnow-action@v1`): дополнительный
  `git push --force <sha>:refs/tags/v1` в шаге сплита только для `action`.
- `action.yml`: `name: 'IndexNow submit (indexnowkit)'` (уникальность проверить на Marketplace перед публикацией), `description`,
  `branding: { icon: 'send', color: 'blue' }`, `runs: { using: docker, image: 'docker://ghcr.io/indexnowkit/indexnow:<ver>' }`
  (пин на версию CLI; бамп — часть релиза CLI), `args: ['/entrypoint.sh']`? — нет: `entrypoint.sh` внутри образа не нужен, Action
  использует `env` + свой `entrypoint.sh`, который лежит в репо действия и монтируется? Docker-действие не монтирует свой
  каталог; поэтому **entrypoint живёт в образе**: `/usr/local/bin/indexnow-action` (тонкий shell в `packages/cli/docker/`),
  `action.yml` → `runs.entrypoint: /usr/local/bin/indexnow-action`. Он читает `INPUT_*` (GitHub передаёт входы так), кладёт
  `INDEXNOW_*`, зовёт `indexnow check --json` (без `--live`, только конфигурация и ключ-файл — ключ-файл на проде должен быть
  доступен: это главное отличие от конкурентов), затем `indexnow sitemap --json …` или `indexnow submit --json …`, пишет
  `$GITHUB_STEP_SUMMARY` (таблица результата) и `$GITHUB_OUTPUT`.
- Входы: `key` (required), `base-url` (required; хост берётся из него), `key-location`, `engines` (по умолчанию ядра — `api`),
  `sitemap` (URL или путь в checkout; по умолчанию `<base-url>/sitemap.xml`), `urls` (многострочно; если задано — `submit`,
  не `sitemap`), `changed-since`, `new-only` (`true` → нужен `state-path` под `actions/cache`; рецепт в README), `state-path`
  (по умолчанию `.indexnow`), `verify` (`true` = pre-flight включён, `verify.enabled`), `dry-run`, `fail-on-error` (по умолчанию
  `true`: exit ≠ 0 роняет шаг; `false` — notice). Выходы: `submitted`, `skipped`, `failed`, `summary-json` (путь к файлу).
- CI действия: в монорепо джоба `cli / action` — собирает образ локально и запускает `entrypoint` с `INPUT_*` против
  mock-сервера (sitemap-файл из fixtures, `INPUT_DRY-RUN`? — имена входов с дефисом GitHub передаёт как `INPUT_DRY-RUN`; проверить
  в документации при реализации — research-first).

### 3.9. Что меняется в существующих пакетах

- core 0.14.0 (аддитивно): `Config::arrayFromEnv()` (§3.3); `docs/configuration.md` — абзац «Environment: every option, one rule»
  со ссылкой на CLI; README core «Install» — строка про `indexnow` для не-фреймворков.
- sitemap 0.9.0: `SeenStoreInterface`, `SitemapRunner(…, ?SeenStoreInterface $seen)`, `SitemapOptions::$newOnly`,
  `Definitions::sitemap()` `--new-only`, `SitemapServices::runner(…, ?SeenStoreInterface $seen = null)`; README «The command»;
  `docs/adapters.md`; `docs/bc.md` (новый интерфейс — тир Implement); CHANGELOG «Added». Адаптеры не трогать (их конструкторы не
  меняются; опция появляется у всех сама — через `Definitions`; тест бандла/Yii3/Laravel/Yii2 на список опций `sitemap`, если
  есть, обновить ожидание).
- console: без изменений (проверить: `Vocabulary` покрывает пустые `submitSubjects`/`explain`? `count()` и тексты не должны печатать
  пустые имена — grep по использованию `$words->explain`/`submitSubjects` в раннерах; если печатаются в `check`/`submit` — вести как
  «команда отсутствует» — см. §7).
- testing: `ConformanceIdsTest::adapters()` — `cli` в `$libraries` (`:80`); без релиза (тест).
- Монорепо: `ci.yml` (матрица + `cli / phar`, `cli / docker`, `cli / action`), `split.yml` (`cli`, `action`), `taint.yml` (`cli`),
  `bin/docs-collect` (`cli`, Getting started «Any site (CLI, cron, CI)»), `php/README.md` таблица, `bin/release-notes` (edit),
  `tools/box`, `bin/phar`, `AGENTS.md` (строка `bin/phar`).

## 4. BC и версии

| Пакет | Версия | Что |
|---|---|---|
| core | 0.14.0 | `Config::arrayFromEnv()` (Call); ничего не ломается |
| sitemap | 0.9.0 | `SeenStoreInterface` (Implement), `--new-only`, десятый аргумент раннера, `SitemapServices::runner()` растёт именованным |
| cli | 0.1.0 | новый |
| testing | — | правка теста, релиза нет |
| console, verify, history, doctrine, адаптеры | — | не меняются; каскад `core ^0.14` **не нужен** (адаптеры на `^0.13` совместимы с 0.14 по SemVer? нет — `^0.13` не включает 0.14 до 1.0). |

Каскад: `^0.13` не разрешает 0.14 (0.x). Значит либо (a) core **0.13.1** (патч; метод аддитивный, семантически «bugfix» натянут),
либо (b) 0.14.0 с каскадом `core ^0.14` по девяти пакетам и их патч-релизами. Волны K–M уже показали цену каскада (девять тегов).
**Рекомендация: (a) core 0.13.1**, sitemap 0.9.0 требует `core ^0.13.1`; cli требует `core ^0.13.1`, `sitemap ^0.9`. Адаптеры на
`sitemap ^0.8` продолжают жить (0.9 они не подхватят до своего следующего релиза — `--new-only` им и не нужен). Записать в
`core/CHANGELOG.md` под «Added» с пометкой «patch by the cascade rule of the family (spec 19b §4)».

## 5. Тесты

- **cli** (phpunit, `tests/Unit`, `tests/Feature` на `CommandTester` + `Testing\FakeTransport`): `EnvConfigSourceTest` (правило
  блоков на трёх пакетах, `hosts` из env заменяет карту, JSON + env приоритет, неизвестная переменная → warning `check`, отсутствующий
  `--config` → INVALID с путём); `DotenvTest` (реальное окружение выигрывает, `--no-env-file`, явный отсутствующий файл → ошибка);
  `SqliteCacheTest` (PSR-16: get/set/delete/has/multiple/TTL на `FrozenClock`, невалидный ключ → `InvalidArgumentException`,
  просроченное не отдаётся); `SqliteSeenStoreTest` (новое / изменённый lastmod / без lastmod / повтор → пусто; `remember()` в
  транзакции); `StateTest` (создание каталога 0700, `memory`, read-only каталог → `ConfigurationException` с подсказкой);
  `KeyFileCommandTest` (два хоста, `key_location` в подкаталоге, `--dry-run`, unchanged, docroot нет → ошибка);
  `ApplicationTest` (`list` — восемь команд, ничего не построено: транспорт не создан — `FakeTransport` без вызовов; `--version`);
  `CheckCommandTest` (`--json` валиден по `console/docs/check.schema.json`; строки `debounce.store: state (…)`, `history.store`,
  `state:`); `SitemapCommandTest` (`--new-only` дважды: второй прогон 0 URL; с `--dry-run` не запоминает; `--changed-since` +
  `--new-only`); `SubmitCommandTest`, `HistoryCommandTest`, `StatusCommandTest` (после `submit` история из состояния);
  `ReadmeAssertions` из testing на README (как у пакетов). Conformance-иды не нужны (не адаптер).
- **sitemap**: `SitemapRunnerTest` — `--new-only` без стора (INVALID + текст), со стором (in-memory double в `tests/Support`),
  `remember()` не зовётся при `--dry-run` и для упавших батчей; `DefinitionsTest` — опция в списке.
- **core**: `arrayFromEnv()` — равенство с `fromEnv()` на полном наборе переменных, «только заданное» (пустой env → `[]`).
- **CI**: cli в матрице (8.2–8.5 highest, 8.2 lowest); `cli / phar` (§3.6), `cli / docker` (§3.7), `cli / action` (§3.8);
  `taint.yml` + `psalm.xml` cli (стоки: `key:file` путь docroot, `--config`, `--env-file`, `--state` — подавления с причиной, как
  `--env-file` у console); mutation — нет (composition root; решение 8). Coverage-floor cli — с первой CI-джобы.

## 6. Документация

- `packages/cli/README.md` + `README.ru.md` по шаблону спеки 90: «Who gets notified», Install (composer global / PHAR / Docker —
  три команды), Quick start (`key:generate --write-env`, `key:file /var/www/html`, `check --live`, `sitemap --new-only` в cron —
  одна строка crontab), «Any CMS: Bitrix, WordPress, MODX, OpenCart» (где у каждой лежит sitemap; Битрикс — штатный модуль),
  «Static sites: on deploy» (GitHub Action и GitLab CI пример с образом), «State file» (`.indexnow/state.sqlite`: что внутри,
  `--state memory`, бэкап не нужен), «Configuration» (правило `INDEXNOW_<BLOCK>_<KEY>`, JSON-файл, приоритет), «Commands»
  (восемь, с ссылками на доки пакетов), «Docker», «GitHub Action» (входы/выходы, `actions/cache` рецепт для `new-only`),
  «Limitations» (нет `explain`/`submit-<subject>`: это адаптеры; `ext-intl` в образе нет), «Other packages», «Notes for AI
  assistants», Versioning.
- `packages/cli/docs/`: `configuration.md` (таблица переменных всех блоков — генерировать? `bin/config-table` знает `Config::OPTIONS`
  и `*Config::OPTIONS`: добавить колонку «CLI env» — да, одна колонка), `state.md`, `action.md`, `docker.md`, `bc.md`.
- `packages/cli/action/README.md` — самостоятельный (Marketplace показывает его): пример workflow на 12 строк, входы/выходы,
  чем отличается от других действий (все движки за прогон, проверка ключ-файла до отправки, pre-flight, `new-only`, история).
- core: `docs/configuration.md` абзац; README «Install». console README §Plain PHP → «or install the CLI». sitemap README/
  `docs/adapters.md` про `SeenStoreInterface`. `php/README.md` таблица. Docs-сайт: раздел Getting started «Any site (CLI)».
- Спека 18 §8 — «сделано волной N, спека 19b»; roadmap; семейный `php/CHANGELOG.md` абзац «Wave N».

## 7. Риски и что проверить первым (research-first)

1. **`Vocabulary` с пустыми `explain`/`submitSubjects`** — проверено: `$words->explain`/`submitSubjects` печатают только
   `ExplainRunner:125`, `SubmitSubjectsRunner:74` и `SubmitSubjectsCommand:32`, которых у CLI нет. Риска нет; пустые строки в
   словаре допустимы.
2. **PHAR и `php-http/discovery`** — проверено: `VerifyServices::transport()` (`verify/src/Adapter/VerifyServices.php:84-87`) зовёт
   `TransportFactory::lazy($verify->transportConfig($config), null, ['User-Agent' => …], BODY_LIMIT)` **без** фабрик PSR-17 и без
   клиента → внутри PHAR pre-flight пойдёт через discovery. Два выхода: (a) verify 0.4.1 аддитивно: `transport(…, ?RequestFactoryInterface
   $requestFactory = null, ?StreamFactoryInterface $streamFactory = null, ?ClientInterface $client = null)` и `transportFor()` берёт их
   из `Services` (у графа фабрик нет — только транспорт; значит `Services`/`ServicesBuilder` узел `psr17`? — нет: CLI передаёт явно);
   (b) CLI собирает `VerifyingSubmitter` сам через `VerifyServices::submitterFactory()` с транспортом
   `Psr18Transport::discover($client, $verify->transportConfig($config)->httpTimeout, ['User-Agent' => $verify->userAgent()],
   VerifyConfig::BODY_LIMIT, $req, $stream)`. **Рекомендация (b)**: ни одного нового параметра в verify, discovery в CLI не вызывается
   нигде; тест `cli / phar` это доказывает (`Http\Discovery\Psr18ClientDiscovery` в PHAR без стратегий → исключение, если бы вызвался:
   в тесте PHAR подменить стратегии нельзя — достаточно `grep -c Psr18ClientDiscovery` в трассе? нет: включить в CLI
   `ClassDiscovery::setStrategies([])` в `bin/indexnow` — тогда любой скрытый вызов discovery падает громко, и в PHAR, и в тестах).
3. **`check-requirements` box** берёт `ext-*` из `composer.lock` сборки — `ext-pdo_sqlite` обязателен для CLI, но у PHAR-пользователя
   на shared-хостинге его может не быть → PHAR откажет с понятным текстом (это правильно), README должен это назвать.
4. **Marketplace**: имя уникально; публикация — только руками пользователя (UI, 2FA); `README` действия должен быть в корне зеркала.
5. **GHCR**: первый пуш — приватный пакет; `docker://` в `action.yml` требует публичности — до публикации действия пользователь
   переключает видимость.
6. **Гонка релиза**: `bin/release-notes --create` (монорепо, после `packagist-wait`) против `release.yml` сплита (ассеты). Правило
   §3.6: сплит ждёт релиз до 10 минут, потом создаёт; `release-notes` умеет `edit`.
7. **Dotenv и `$_ENV`**: `variables_order` на некоторых хостингах без `E` — `$_ENV` пуст; `Dotenv` пишет в `$_ENV` и `$_SERVER`
   всегда → `Config::fromEnv()` читает `$_SERVER` (`Config.php:396`) — ок; тест на `variables_order=S`.
8. **Read-only контейнер**: `Spool` умеет память (`sitemap/src/Spool.php`); состояние — `--state memory`; `check` говорит об этом
   (`state: memory (nothing persists between runs)`).
9. **PHP 8.5**: гонять `PHP_VERSION=8.5 bin/ci cli` до пуша (урок волны M).
10. **`INPUT_*` для входов с дефисом** — проверить в документации GitHub при реализации.

## 8. Не делать (рассмотрено)

- Модуль Битрикса — следующая волна; CLI + cron + штатный sitemap покрывает Битрикс сейчас.
- phive/GPG — нужен ключ подписи пользователя; вернуться, когда появятся запросы (`.phar.asc` в релиз — 5 строк в `release.yml`).
- `bin/indexnow` внутри `indexnowkit/console` (вариант спеки 18 §8): console — библиотека адаптеров, ей не нужны dotenv, http-client,
  sqlite; отдельный пакет держит зависимости там, где они используются (решение 1).
- `explain`/`submit-<subject>`: нет ORM — нет субъектов. Для «объяснить URL» есть `check --sample <url>` (verify).
- `symfony/cache` ради PSR-16: своя таблица (решение 4).
- Homebrew tap, npm-обёртка, Windows-инсталлятор, composite-вариант действия (setup-php + скачивание PHAR): второй путь = второе
  поведение.
- RSS/Atom как источник (у bojieyang есть): sitemap-протокол — норма; RSS — по спросу, в `SitemapReader` отдельной волной.

## 9. `[решение]` — рекомендации (утверждает пользователь)

1. **Отдельный пакет `indexnowkit/cli`**, бинарник `indexnow`, сплит `php-cli` — (a) да. Альтернатива (b) `bin` в console — нет.
2. **`Config::arrayFromEnv()` в core** — да (иначе CLI дублирует 40 строк разбора).
3. **Умолчания CLI**: `debounce.store = state`, `history.store = pdo` над файлом состояния — да.
4. **PSR-16 над sqlite своей реализацией** (не `symfony/cache`) — да.
5. **`--new-only` и `SeenStoreInterface` в sitemap 0.9.0** — да; адаптеры получают опцию с честным отказом, стор — позже.
6. **Action = Docker-действие на образе GHCR**, entrypoint в образе, источник в монорепо, зеркало сплитом с тегами `vX.Y.Z` + `vX` — да.
7. **`symfony/dotenv`** для `.env` — да; JSON-файл `--config` — да.
8. **Taint для cli — да; mutation — нет.**
9. **Версии**: core 0.13.1 (без каскада), sitemap 0.9.0, cli 0.1.0, action 1.0.0 — (a). Альтернатива (b) core 0.14.0 + каскад ×9 — нет.
10. **Имя действия** — «IndexNow submit (indexnowkit)»; пользователь может выбрать другое до публикации.

## 10. Definition of Done

- `composer global require indexnowkit/cli` → `indexnow check` работает в пустом каталоге с одним `INDEXNOW_KEY` и `INDEXNOW_BASE_URL`;
  `indexnow key:generate --write-env && indexnow key:file /tmp/docroot && indexnow check --live` (против mock-сервера) зелёные.
- `indexnow sitemap --new-only` дважды подряд: второй прогон «0 new or changed»; после правки `lastmod` одной записи — 1.
- `php indexnow.phar --version` печатает версию тега; `cli / phar`, `cli / docker`, `cli / action` в CI зелёные; образ запускается
  под непривилегированным пользователем; `docker run … check --json` валиден по схеме.
- `action.yml` в корне зеркала `indexnowkit/indexnow-action`, `uses: indexnowkit/indexnow-action@v1` на тестовом репозитории
  отправляет sitemap-файл на mock… — нет, на живой `api.indexnow.org` с `dry-run: true` (dry-run ничего не шлёт): шаг зелёный,
  summary содержит таблицу.
- Гейт семьи зелёный: `bin/ci` ×12 highest, lowest 8.2 ×12, `PHP_VERSION=8.5 bin/ci cli`, taint core/console/sitemap/verify/history/cli,
  mutation прежних пяти не ниже пола, `bin/cs`, composer validate ×12, `config-table --check`, mkdocs --strict.
- Документы §6; спека 18 §8 закрыта; память; пуш/теги — по «действуй», порядок: core 0.13.1 → sitemap 0.9.0 → cli 0.1.0 →
  (пользователь: Packagist `indexnowkit/cli`, GHCR public) → `action@1.0.0` → (пользователь: Marketplace).
