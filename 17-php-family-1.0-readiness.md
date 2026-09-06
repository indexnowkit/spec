# 17. Семейство PHP к 1.0: состав core, DX, UX, SEO, дистрибуция

Статус: спецификация v2 (после двух адверсальных ревью 2026-09-05: реализуемость — 33 находки, дизайн — 60). Волны 0a + hotfix,
0b и D выполнены 2026-09-05 (§11, §12, §13), E и F — 2026-09-06 (§14, §15; F выпущена 2026-09-06: core 0.9.0, console 0.3.0, testing 0.2.0, verify 0.1.0, history 0.1.0, sitemap 0.5.0, doctrine 0.7.0, symfony-bundle 0.10.0, laravel 0.11.0, yii2 0.9.0). Основание: пять аудитов по линзам «состав core», «DX разработчика и AI-ассистента», «UX владельца сайта»,
«SEO-корректность», «дистрибуция». Исходное состояние: core 0.5.1, sitemap 0.1.1, doctrine 0.3.1, symfony-bundle 0.6.1,
laravel 0.7.0, yii2 0.5.0.

Цель: семейство готово к 1.0 и удобно трём аудиториям — автору адаптера, разработчику приложения (в том числе через
AI-ассистента) и владельцу сайта/SEO-специалисту. Yii3 и Битрикс начинаются после этой спеки на её базе.

## 0. Сводка

| Линза | Сейчас | Главный разрыв |
|---|---|---|
| Состав core | 125 файлов, 10.2k строк; `symfony/console` и `phpunit` в `suggest`, хотя 12 файлов `src/` их импортируют | `Console\*` и `Testing\Conformance\*` — фичи, не ядро |
| DX человека | quickstart 7, ошибки 8, DSL 7, docs 8, testing 8 | три README-дефекта; Yii2-доки в 4–6 раз тоньше; ключи `router.locales`/`router.languages` дают только warning в логе |
| DX AI-ассистента | 3 | нет машиночитаемых артефактов, которые реально доезжают до ассистента |
| UX/эксплуатация | ключи 7, наблюдаемость 6, check 7, стейджинг **4**, CLI 8 | стейджинг с боевым ключом шлёт боевые запросы, `check` молчит; нет `--json`/`--strict`; нет истории |
| SEO | протокол 9, вред **5**, удаление 8, дедуп 8, честность 7 | ни слова про `noindex`/robots/canonical; «≠ индексация» не сказано; дефект дебаунса при нескольких движках |
| Дистрибуция | metadata 8, GitHub 5, discoverability 5, security 6 | topics/LICENSE/CoC/шаблоны/Dependabot; витрина Packagist отстаёт |

Принципы (продолжают спеку 16):

1. **Фича — потребитель core.** Что не нужно каждому адаптеру в веб-запросе — отдельный пакет с сохранением FQCN
   через PSR-4. Core не импортирует `Symfony\Component\Console\` и `PHPUnit\`; `symfony/http-client` остаётся
   опциональной целью дискавери в `Psr18Transport` (это транспорт, не фича).
2. **Машиночитаемо по умолчанию.** Структурный вывод у диагностики, схема у конфига, README-раздел для ассистентов.
3. **Честно.** IndexNow — уведомление, не индексация; Google не участвует; где смотреть результат — сказано.
4. **Безопасно по умолчанию.** Непроизводственное окружение не отправляет по ошибке; `check` видит всё, что вредит.
5. **Что разрешено патчем/минором по `bc.md` — делается сразу**, не ждёт большой волны.

Волны и версии:

| Волна | core | Содержание | Срок | Риск |
|---|---|---|---|---|
| 0a + hotfix | **0.6.0** | §2: стейджинг-проверка, дефект дебаунса, `PREVIOUS_KEY`, два `Engine`, тексты ошибок, `DebounceStoreCheck` в бандле, README-дефекты, гигиена GitHub/Packagist | дни | низкий |
| 0b | — | §3: документация, AI-артефакты, docs-сайт, RU, паритет Yii2 | недели, параллельно с D/E | нулевой |
| D | **0.7.0** | §4: пакеты `testing`, `console`; `Adapter\OptionalPackage`; перемещения | дни | низкий (механика) |
| E | **0.8.0** | §5: `check --json/--strict` и новые проверки, ротация, счётчик 403, `SubmissionStoreInterface`, канонизация, `Condition`, `config --json` | 1–2 недели | средний |
| F | **0.9.0** (аддитивно) | §6: `verify` 0.1.0, затем `history` 0.1.0 — **выполнена 2026-09-06** (§15) | после E | изолирован |
| 1.0 | 0.9 → 1.0 | §7: один минор без breaking, критерии выполнены | — | — |

## 1. Целевая карта пакетов

```
L0  indexnowkit/core        протокол, модель правил, адаптерный кит, 4 тестовых двойника без PHPUnit
 ├─ L1 indexnowkit/console  раннеры CLI и Definitions             require: core, symfony/console ^6.4|^7|^8
 ├─ L1 indexnowkit/testing  conformance-киты и assertion-хелперы   require: core; suggest phpunit ^11
 ├─ L1 indexnowkit/sitemap  чтение sitemap + команда               require: core, console, ext-xmlreader
 ├─ L1 indexnowkit/verify   pre-flight проверка URL, check --sample require: core; suggest console      (волна F)
 └─ L1 indexnowkit/history  история отправок                       require: core; suggest console      (волна F)
L2  indexnowkit/doctrine    require: core; dev: testing
    indexnowkit/{symfony-bundle,laravel,yii2,yii3,bitrix}  require: core, console; suggest: sitemap, verify, history; dev: testing
```

Решения по составу core:

| Кандидат | Строк | Решение | Основание |
|---|---:|---|---|
| `Console\*` кроме `SubmitterFactory*`, `ResultSummary` | ~1075 | **вынести** → `indexnowkit/console`, FQCN сохраняются | core продаётся как framework-agnostic; `require symfony/console` в core тянул бы 4–6 пакетов и матрицу трёх мажоров Symfony в plain-PHP и doctrine (ноль упоминаний `Console\`). Рассмотрено и отклонено: `symfony/console` в `require` core; оставить в core `Definitions`/`Vocabulary`/`ExitCode` (используются только внутри `Console\`) |
| `Console\SubmitterFactory`, `SubmitterFactoryInterface` | 71 | **переместить** → `Adapter\` | зовутся из `Adapter\Services`; слой 2 не должен знать про CLI |
| `Console\ResultSummary` | 64 | **переместить** → `IndexNowKit\Submission\ResultSummary` | агрегат отправок, не протокол; `Submission\` в волне E получает `SubmissionStoreInterface` |
| `Testing\Conformance\*`, `KeyFileAssertions`, `CheckOutputAssertions` | 760 | **вынести** → `indexnowkit/testing`; хелперы → `Testing\Conformance\{KeyFileAssertions, CheckOutputAssertions}` | расширяют PHPUnit из `suggest`; ноль потребителей в `src`; `OrmConformanceTestCase` 453 строки в прод-vendor. Хелперы сегодня в тире Call — в CHANGELOG это «изъятие из стабильного API до 1.0», не «одна строка `use`» |
| `Testing\{FakeTransport, ArrayLogger, FrozenClock, RecordingDispatcher}` | 212 | оставить | реализации интерфейсов core без PHPUnit; спека 10 обещает их в основном пакете |
| `Adapter\Services`/`ServicesBuilder` | 528 | оставить до Yii3; критерий — форма, не факт использования: Yii3 обходится **без новых методов** `ServicesBuilder`; потребовалось расширение — код возвращается в yii2 к 0.9 | один потребитель |
| `Transaction\VerifyingStaging` | 155 | тот же критерий с Yii3; иначе → yii2 | один потребитель |
| `Check\*`, `Transaction\*` (остальное), `Key\*`, `Retry\*`, `Url\Punycode`, `Attribute\Param\*`, `Hook\*` | — | оставить | ноль внешних зависимостей, реальные потребители или протокольный шов |
| `Adapter\OptionalPackage` | +40 | **добавить** в core | две копии `SitemapSupport` (laravel, yii2) + предикат в бандле |
| `SitemapConfig::loadOrDisabled()` | +25 | **добавить** в sitemap | две копии «невалидный блок → critical → disabled» (laravel, yii2); бандл не использует: у DI-фабрики нет логгера в момент вызова, невалидный блок падает при сборке контейнера — так и остаётся |

Core после волны D: ~106 файлов, ~8.3k строк. Зависимости core (`php-http/discovery`, `psr/*`) обязательны и остаются;
из `suggest` уходят `symfony/console` и `phpunit/phpunit` (`phpunit` остаётся в `require-dev` — у core свои тесты).
`sitemap` требует `console` (его `SitemapRunner` импортирует шесть классов console); `symfony/console` уходит из
`suggest` sitemap. Не дробить `Check`, `Key`, `Transaction`, `Attribute\Param`, по слоям «protocol/rules» — контрпримеры
в отчёте линзы, решение окончательное.

## 2. Волна 0a + hotfix (core 0.6.0, адаптеры минорами)

Всё, что `bc.md` разрешает минором без breaking: рост `Engine`/`Reason`, аддитивные поля и параметры с дефолтом,
тексты сообщений («not API»), новые строки диагностики. Делается первым, потому что содержит единственную находку с
необратимыми последствиями.

### 2.1. Стейджинг с боевым ключом (`Check\Checker`)

`Config` получает `public readonly bool $dryRunExplicit` (заполняется в `fromArray()` из `array_key_exists('dry_run',
$data)`, в `with()`/`withDryRun()` — `true`); `$dryRun` остаётся `bool` (смена типа = breaking для всех читателей).
`Checker::run()` после проверки `dry_run`:

| Состояние | Строка |
|---|---|
| `environment === null` (plain PHP без `APP_ENV`/`INDEXNOW_ENV`) | ничего: судить не о чём |
| не production, `enabled`, ключ есть, `dryRun === false`, `dryRunExplicit === false` | **error** `environment "staging" is not in production_environments but dry_run is off: changes WILL be sent to search engines under key ab12****. Set INDEXNOW_DRY_RUN=1 or INDEXNOW_ENABLED=0 outside production, or set dry_run: false explicitly if this environment submits on purpose.` |
| то же, но `dryRunExplicit === true` | **warning** `... dry_run is explicitly false, assuming this environment submits on purpose.` |
| всегда, когда `environment !== null` | ok/warning `environment: staging (not in production_environments: prod, production)` |

Выход для preview-окружений — явный `dry_run: false`; `production_environments` для этого не используется (он управляет
авто-dry_run и текстами, а не «кому можно слать»). Тест на четыре состояния.

### 2.2. Прочий код hotfix

- **Дефект дебаунса при нескольких движках** (`Submitter.php:74-83`): не маркировать URL, для которых есть retryable-результат:
  `array_values(array_diff(Result::urlsWhere($results, isSuccess), Result::retryableUrls($results)))`. Перманентный отказ
  (403/422) одного движка при успехе другого маркирует — ретраем не лечится, дебаунс не наказывает успешный движок.
  `engines: ['api']`, dry_run, невалидные URL — поведение не меняется. Тесты `SubmitterTest` с частичным успехом.
- `Config::fromEnv()` читает `INDEXNOW_PREVIOUS_KEY`; таблица env в `configuration.md`.
- `Engine::InternetArchive = 'internetarchive'` (`https://internetarchive.indexnow.org/indexnow`), `Engine::Amazon = 'amazon'`
  (`https://indexnow.amazonbot.amazon/indexnow`); источник — `https://api.indexnow.org/searchengines.json` (снимок
  2026-09-05), список фиксируется в `docs/spec/01-protocol.md:8-22`. Хост Яндекса (`yandex.com` vs `www.yandex.com`) —
  решение пользователя после реального запроса (§10).
- **Тексты ошибок** (правило: факт + допустимое + как исправить): `Config.php:192` (resolver — два сообщения с
  фактами), `:198` (key_prefix — расшифровать ограничение), `:214` (retry — пять отдельных проверок);
  `Engine.php:61` (добавить «or an alias from engine_aliases»); `ClassNameResolver:46,49` (где искали; для чего нужен
  класс); `ParamExtractor:88` (что допустимо), `:129` (перечислить перебранные имена); `ModelLoader:80`,
  `ActiveRecordLoader:70` (какой базовый класс); yii2 `IndexNowObserver:229` (что делать); `Checker:128` (начало тела:
  «a 200 answer with HTML usually means a catch-all route matched first»).
- `Retry\RetryingSubmitter` docblock: путь для sync/CLI; очереди используют `WorkerOutcome`.
- `DebounceStoreCheck` в бандле с probe над `Psr16Cache` (паритет с Laravel/Yii2).
- `check` финал: после `IndexNow is ready.` — `Next: annotate a class with #[IndexNow(...)], or send one URL now:
  {cli} {submit} https://…` (поля `Vocabulary::$cli/$submit` уже есть).

### 2.3. README и документы (только тексты)

| Файл | Дефект | Правка |
|---|---|---|
| `laravel/README.md:10`, `README.ru.md`, `CONTRIBUTING.md:12` | «Laravel 11» при `^12 \|\| ^13` | «12 \| 13» |
| `laravel/README.md:86-92`, `README.ru.md:90` | сниппет без `use IndexNowKit\Attribute\{IndexNow, IndexNowDefaults, RuleSet}` | добавить импорты |
| `yii2/README.md:22-41` | команды до конфигурации компонента (контроллер регистрируется в `bootstrap()`) | блоки местами |
| `laravel/README.md` у фасада | два класса с именем `IndexNowKit` | абзац «фасад `Laravel\Facades\IndexNowKit`; ядро `IndexNowKit\IndexNowKit` инжектится по типу». Алиасы `submitEntity` **не вводятся** (противоречат `Vocabulary`, замораживали бы 4 метода) |
| `core/README.md:210` | `serve_key_file` в пользовательском README | убрать; в troubleshooting бандла оставить (пользователь ищет по старому ключу) |
| `sitemap/README.md` | нет Google-абзаца | абзац из core README + строка про `lastmod = now()` |
| все README «Who gets notified» | нет Internet Archive и Amazon | список из `searchengines.json` |
| все README, шапка | нет пути к issues при выключенных Issues на сплитах | строка «Issues and pull requests: github.com/indexnowkit/php» рядом с «Русская версия» |
| все README | нет «Why this over X» | раздел из спеки 90 (атрибут + дебаунс + post-commit + 429 backoff + общий core) |
| `docs/bc.md` в doctrine, symfony-bundle, laravel, yii2 | нет | что стабильно: конфиг-ключи, команды, service id/binding'и/свойства; ссылка на тиры core |
| §5.2/§5.3 SEO-тексты волны 0b, которые не требуют кода | — | «≠ индексация», BWT/Я.Вебмастер, 410/301/404, полный sitemap-прогон — можно и в 0a, если успевают |

### 2.4. GitHub и Packagist

- **Витрина Packagist.** `repo.packagist.org/p2/indexnowkit/<pkg>.json` актуален (проверено 2026-09-05: core 0.5.1,
  symfony-bundle 0.6.1), `packagist.org/packages/<pkg>.json` и страницы отстают. Пользователь нажимает «Update» на трёх
  страницах. `php/bin/packagist-check <pkg>`: последний тег сплита vs `p2` vs `packages/*.json`, расхождение = warning;
  вызывается в конце `bin/tag`.
- **Topics**: `php-sitemap` (`indexnow sitemap seo php`), `php-laravel` (`indexnow laravel eloquent seo php`), `php-yii2`
  (`indexnow yii2 activerecord seo php`); `homepage` у всех сплитов = монорепо (до сайта).
- **`indexnowkit/spec`: LICENSE** — MIT (дефолт; спека содержит код, фикстуры и mock-сервер, которые копируются в
  MIT-порты); вариант двойной лицензии CC-BY-4.0 текст + MIT код — решение пользователя (§10).
- **Read-only сплиты:** выключить Issues, Wiki, Projects (`gh repo edit …`) — после строки в README (§2.3).
- **Монорепо `php/`:** `.github/dependabot.yml` с `groups` по директории (composer ×6 + github-actions, недельно,
  `open-pull-requests-limit: 3`); шаг `composer audit` в `ci.yml` для каждого пакета; `roave/security-advisories:
  dev-latest` в `require-dev` каждого пакета; для `sitemap` (недоверенный XML) — scheduled-джоб Psalm
  `--taint-analysis`. **CodeQL не поддерживает PHP — не использовать.**
- **`indexnowkit/.github`** (низкий приоритет): README организации (что это, таблица пакетов), `CODE_OF_CONDUCT.md`
  (Contributor Covenant 2.1). Issue/PR-шаблоны — в монорепо `php/.github/` (bug: пакет, версия, фреймворк, вывод
  `indexnow:check --json`, репродукция; feature; PR-чеклист из CONTRIBUTING). `FUNDING.yml` не создаётся; упоминание
  спонсорства убрать из спеки 90.
- **composer.json:** `authors` в doctrine/symfony-bundle/laravel; `config.allow-plugins: {"php-http/discovery": false}`
  во всех адаптерах; keywords `psr-18`, `psr-3` у адаптеров.
- **Бейджи** (единый порядок в шести README): version, downloads, CI, conformance, **`coverage ≥ NN% enforced`**
  (статический, NN = floor из `tests/coverage-floor.txt`, меняется с floor'ом), `phpstan level 9`, PHP, фреймворк,
  license. Живой процент через Codecov — решение пользователя (§10).
- **SECURITY.md** ×7: «acknowledged within 5 business days; fix or mitigation plan within 30 days».
- **Flex.** Private recipe endpoint — **не** «автоустановка»: требует сгенерированного `index.json` + `vendor.package.
  version.json`, URL вида `api.github.com/repos/…/contents/index.json` и ручного opt-in в composer.json пользователя.
  Не делать; ручная установка остаётся в README; PR в `recipes-contrib` — после трекшна и решения по trademark (§7).

### 2.5. Релизы волны 0a

`core@0.6.0` → `symfony-bundle@0.7.0` → `laravel@0.8.0` → `yii2@0.6.0` → `doctrine@0.4.0` (только `core ^0.6`) →
`sitemap@0.2.0` (`core ^0.6`). Гейт: тест четырёх состояний стейджинга; тест дебаунса; `check` в staging-конфиге
адаптера с ключом даёт exit 1.

## 3. Волна 0b: документация, AI-артефакты, docs-сайт (параллельно, без релизов кода)

### 3.1. Что доезжает до AI-ассистента потребителя

| Канал | Доезжает | Решение |
|---|---|---|
| `AGENTS.md` в vendor-пакете | нет (агенты читают корень рабочего репо, `vendor/` исключён; Packagist рендерит README) | **один** `AGENTS.md` в корне монорепо для контрибьюторов (bin/*, коммиты, спека, гейты) |
| README пакета | да (Packagist, поиск, индексаторы) | раздел `## Notes for AI assistants` (~15 строк) в README каждого пакета: имя пакета, минимальный сниппет с полными `use`, команда проверки, **ловушки**: `dispatch: auto` есть в Symfony/Yii2 и нет в Laravel; `router.locales` (Laravel) vs `router.languages` (Yii2); `url:` — имя аксессора, `urls:` — литералы; строка в `when` truthy — нужен `Equals`; `submitEntity`/`submitModel`/`submitRecord`; фасад vs ядро |
| Laravel Boost | да | `laravel/resources/boost/guidelines/core.blade.php` — **ровно один файл**, 30–50 строк, содержимое = README-раздел в формате Boost |
| `.phpstorm.meta.php` в core | да (PhpStorm сканирует vendor) | `expectedArguments` для `IndexNow::__construct` `locales` (`current`, `all`), `events` (`created`, `updated`, `deleted`) |
| `context7.json` | только после submit репозитория в Context7 | файл в корне (`folders`, `excludeFolders: vendor, tests, docs/plans, docs/spec`, `rules` = ловушки) + шаг «submit `indexnowkit/php` на context7.com» |
| docs-сайт + `llms.txt`/`llms-full.txt` | да | §3.2 |

Тесты: `ReadmeAiNotesTest` в каждом адаптере — раздел существует, упомянутые команды есть в `Definitions`, ключи — в
`OPTIONS`/дереве бандла. `ReadmeQuickstartTest`: канонический сниппет — **файл** `tests/Readme/QuickstartModel.php`
(компилируется, phpstan, IDE-рефакторинг), тест (а) утверждает, что README содержит его текст дословно (golden),
(б) прогоняет фикстуру через `FakeTransport` в поднятом приложении. Никакого `eval`.

`bin/config-table`: генерирует таблицу «ключ × дефолт × Symfony/Laravel/Yii2» из `Config::OPTIONS`,
`Sitemap\SitemapConfig::OPTIONS`, `Laravel\Config\ConfigFactory::LARAVEL_OPTIONS`, `Yii2\Config\ConfigFactory::YII_OPTIONS`
и обхода `IndexNowKitConfiguration::getConfigTreeBuilder()`; результат — `core/docs/configuration.md` (раздел
«One concept, three keys») и сайт; тест сравнивает файл с генерацией.

### 3.2. Docs-сайт

MkDocs Material на GitHub Pages (`indexnowkit.github.io/php`), workflow `docs.yml`. Сборка скриптом `bin/docs-collect`
(не `mkdocs-monorepo-plugin` — он не переписывает ссылки): README пакета → `index.md` раздела с переписыванием
`](docs/x.md)` → `](x.md)`; абсолютные `github.com/indexnowkit/php/blob/main/packages/<pkg>/docs/<f>.md` →
внутрисайтовые; полный `nav` генерируется. `mkdocs build --strict` в CI; внешние ссылки — `lychee`. Разделы: Getting
started (Symfony / Laravel / Yii2 / plain PHP) → Attribute reference → Configuration (таблица ключей) → Operations
(prod checklist первым) → Testing → Adapters → Packages → Compatibility. `llms.txt` (оглавление) и `llms-full.txt`
(конкатенация) в корне сайта. `homepage` в composer.json и на GitHub → сайт. Домен — решение пользователя (§10).

### 3.3. SEO-честность и SEO-рекомендации

- «IndexNow ≠ индексация» — абзац в шести README после «Who gets notified» и в `operations.md`.
- Где смотреть результат: Bing Webmaster Tools → IndexNow Insights; Яндекс.Вебмастер → Переобход страниц; метрика
  успеха — доля отправленных URL в индексе за N дней.
- «Deleted pages: what your site must return» в `operations.md` + три строки в README: 410 навсегда, 301 + оба URL для
  замен, 404 временно; soft-404 и редирект на главную — вред.
- «What not to submit» (`operations.md`, `attribute-reference.md` «Anti-patterns»): `noindex`/`X-Robots-Tag`, robots.txt,
  неканонические URL, 3xx/4xx/5xx, черновики; что защищает сейчас и что даст `verify`.
- Sitemap: полный прогон без `--changed-since` — разовый; описание `--force` дополнить; строка про `lastmod`; код (warning
  в рантайме при > `batch.max_urls` URL без `--changed-since`) — волна E.
- Bing URL Submission API и Google Indexing API — «другое» (два предложения в README core).
- hreflang-кластер через `via:` (`multi-domain.md`); `batch.max_urls` — потолок, не цель.

### 3.4. Эксплуатационная документация

Prod checklist — первый раздел `operations.md` со ссылками из README (ключ и base_url; `check --strict` зелёный;
`strict_hosts`; общий `debounce.store`; очередь под мониторингом; стейджинг: `INDEXNOW_DRY_RUN=1`/`ENABLED=0` и
`key_file.enabled: false`; алерты на три строки; `check --strict` в деплое; `cache_max_age <= 300` и CDN; `previous_key`
снят после ротации). Сценарии «отправились URL стейджинга», «дубли при `memory` и нескольких воркерах» в troubleshooting
трёх адаптеров. Оговорка про эскалацию 403 per-process (до волны E). «Ключ не уходит в query string». Четыре правила
мониторинга + фильтр Sentry. Yii2-паритет: `configuration.md` (полная таблица), `testing.md` (verify-on-commit), новый
`multi-domain.md`, 422/429/дубли в troubleshooting. www/apex — абзац в `multi-domain.md` каждого адаптера. RU-перевод:
`attribute-reference.md`, `configuration.md`, `*/docs/troubleshooting.md` (объём — решение пользователя, §10).
`core/docs/testing.md`: mock-server теперь в пакете `testing` (после волны D).

## 4. Волна D: состав core (core 0.7.0)

### 4.1. Пакет `indexnowkit/testing` 0.1.0

- `packages/testing`, autoload `IndexNowKit\Testing\Conformance\` → `src/` (точный непересекающийся префикс; Composer берёт
  самый длинный совпавший — предупреждений нет). Переезжают `CoreConformanceTestCase`, `OrmConformanceTestCase` (FQCN без
  изменений), `KeyFileAssertions`, `CheckOutputAssertions` → `IndexNowKit\Testing\Conformance\{…}`.
- Mock-server роутер core (`tests/Support/mock-server`) → `testing/resources/mock-server/` (документируется путь в vendor).
- Тесты core, использующие переехавшее (`core/tests/Unit/Testing/AssertionsTest.php`, conformance-прогоны core через
  кит), переезжают в `testing/tests` (пакет требует core — цикла нет); `core/tests/Unit/Adapter/TwentyMinuteAdapterTest.php:142`
  переходит на локальную ассерцию. **Core не получает `require-dev: indexnowkit/testing`** (иначе бутстрап-цикл: сплит
  core с алиасом `0.6.x-dev` не удовлетворит `^0.7` до тега).
- `composer.json`: `require: php ^8.2, indexnowkit/core ^0.7`; `require-dev`/`suggest: phpunit/phpunit ^11`;
  **`extra.branch-alias.dev-main: 0.1.x-dev` обязателен** — `bin/link.php:41-45` падает без него и валит весь CI;
  `minimum-stability dev + prefer-stable`, scripts `ci:install:*`, `.github/workflows/ci.yml`, SECURITY, CHANGELOG, README EN/RU.
- Адаптеры: `require-dev: indexnowkit/testing ^0.1`; core: `phpunit` уходит из `suggest`, остаётся в `require-dev`.

### 4.2. Пакет `indexnowkit/console` 0.1.0

- `packages/console`, autoload `IndexNowKit\Console\` → `src/`; 16 файлов: `CheckRunner`, `ExplainRunner`,
  `KeyGenerateRunner`, `SubmitRunner`, `SubmitSubjectsRunner`, `ResultRenderer`, `ResultFormatterInterface`,
  `CommandDefinition`, `ArgumentDefinition`, `OptionDefinition`, `Definitions`, `Vocabulary`, `ExitCode`,
  `ClassNameResolver`, `SubjectLoaderInterface`, `SubmitSubjectsOptions`. FQCN сохраняются.
- До переноса в core: `Console\SubmitterFactory`, `SubmitterFactoryInterface` → `Adapter\`; `Console\ResultSummary` →
  `Submission\ResultSummary` (breaking, миграция — `use`). `Adapter\Services::submitterFactory()` возвращает
  `Adapter\SubmitterFactoryInterface`. После этого `Console\` вне `Console/` в core не встречается (проверено: только
  `Adapter/Services.php:19-20` и два комментария).
- `composer.json`: `require: php ^8.2, indexnowkit/core ^0.7, symfony/console ^6.4 || ^7.0 || ^8.0`; `branch-alias 0.1.x-dev`.
- Адаптеры symfony-bundle, laravel, yii2 и sitemap: `require: indexnowkit/console ^0.1`; doctrine — только `core ^0.7`.
  core: `symfony/console` уходит из `suggest` и `require-dev`; sitemap: из `suggest`. Тиры `Console\*` → `console/docs/bc.md`.
- Гейт: `grep -rl 'Symfony\\Component\\Console\|PHPUnit\\' packages/core/src packages/sitemap/src` пуст.

### 4.3. `Adapter\OptionalPackage` и `SitemapConfig::loadOrDisabled()` (sitemap 0.3.0)

```php
namespace IndexNowKit\Adapter;

final class OptionalPackage
{
    /**
     * @param class-string $marker   класс, существование которого означает «пакет установлен»
     * @param ?bool        $installed переопределение (тесты, compile-time бандла); null = class_exists($marker)
     */
    public function __construct(public readonly string $package, public readonly string $marker, public readonly string $feature, private readonly ?bool $installed = null) {}
    public function installed(): bool;
    public function notInstalledMessage(): string;   // 'indexnowkit/sitemap is not installed: composer require indexnowkit/sitemap'
    /** @param array<string,mixed> $block @param array<string,mixed> $defaults */
    public function checkLine(array $block, array $defaults = []): string;
    /** уровень: ok — не установлен и блок пуст/дефолтный; warning — блок настроен, но игнорируется */
    public function check(array $block, array $defaults = []): Check\StaticCheck;
}
```

Никакой статики. Переопределение приходит из адаптера: бандл — существующие параметры `IndexNowKitLoader`/
`IndexNowKitConfiguration` (`?bool $sitemapInstalled`); Laravel — тесты биндят экземпляр в контейнер до `boot()`
(`$app->instance(SitemapPackage::class, …)`), провайдер берёт из контейнера, если есть; Yii2 — свойство компонента
`sitemapInstalled: ?bool`. Классы `SitemapSupport` удаляются; `yii2/tests/Feature/SitemapNotInstalledTest.php:70`
переходит на `checkLine([], [])`. `SitemapConfig::loadOrDisabled(array $block, LoggerInterface $logger, string
$checkCommand): SitemapConfig` заменяет копии в laravel/yii2 (унифицирует текст с `(run "{check}")`); бандл не трогается.

### 4.4. Релизы и инфраструктура волны D

Порядок: **регистрация `testing` и `console` на Packagist до первого пуша `main` адаптеров** (иначе `packagist-wait-main`
получит 404 и адаптеры уйдут в сплиты с неразрешимой зависимостью): репо `php-testing`, `php-console`, deploy-keys,
секреты `SPLIT_SSH_KEY_TESTING`/`SPLIT_SSH_KEY_CONSOLE`; `split.yml` стадии `core → {testing, console} → sitemap → адаптеры`;
`tools-push-splits.sh:16-23` и `ci.yml:26,30-38` — списки пакетов; `php/README.md`. Гейт до первого тега: зелёный
прогон всех адаптеров на `core@dev-main + console@dev-main + testing@dev-main` через `bin/link`. Два выноса — два
ревертируемых коммита. Теги: `core@0.7.0` → `testing@0.1.0` → `console@0.1.0` → `sitemap@0.3.0` → `doctrine@0.5.0` →
`symfony-bundle@0.8.0` → `laravel@0.9.0` → `yii2@0.7.0`.

## 5. Волна E: эксплуатация и SEO (core 0.8.0, console 0.2.0)

### 5.1. `indexnow:check` как healthcheck

Модель: `Check\CheckItem` получает `?string $code = null, ?string $host = null` (appended); `CheckReport::ok/warning/error(string
$message, ?string $code = null, ?string $host = null)`; все проверки core и адаптеров получают стабильные `code`
(`config.dry_run`, `environment.non_production_submits`, `key_file.status`, `key_file.content_type`, `debounce.store`, …) —
`code` для `check` то же, что `Reason` для результатов: тексты не API, коды — API (перечень в `docs/check-codes.md`).

| Изменение | Где | Суть |
|---|---|---|
| `--json` | console: `Definitions::check()`, `CheckRunner` | схема `docs/check.schema.json` в репо, тест-фикстура; `{"status":"ok\|warning\|error","environment":?string,"items":[{"level","code","message","host":?string}]}`; `hosts` отдельно не выделяется — группировка по `host` на стороне читателя; `--strict` влияет только на exit-код, не на `status` |
| `--strict` | там же | предупреждения → exit 1 |
| `--host` многократный | там же | список |
| `Content-Type` | `Http\Response` получает `array<string,string> $headers = []` (lowercase) 4-м параметром + `contentType()`; `Psr18Transport::get()` **и** `download()` заполняют; `FakeTransport::onGet` не меняется (тесты передают `new Response(200, $key, headers: […])`) | error только если заголовок есть и не `text/plain`; заголовка нет (кастомный транспорт) → ok «Content-Type unknown (this transport does not expose headers)» |
| `Cache-Control`/`Age` | `Checker` | warning, если `max-age` > `key_file.cache_max_age` («CDN держит файл N с, ротация займёт N с») |
| Пользовательский `http.client` | `Checker` | warning «key file fetched through your http.client; if it follows redirects, a 30x to a catch-all page looks like a 200». Свой транспорт `check` уже без редиректов (`Psr18Transport::createClient`), нового параметра не нужно |
| robots.txt | `Checker` | GET `/robots.txt`, `Disallow` на путь ключевого файла → warning |
| `previous_key` | `Checker::checkHost` | из `Config::$previousKeys[$host] ?? $previousKey` (не через `KeyProviderInterface` — тир Implement); отдаётся ли `/{previous}.txt`: 200 → ok «rotation window open; remove previous_key once check --live is green», иначе warning. Контроль «через сутки» не обещается — времени установки нет |
| `--sample` | **пакет `verify`** (§6.1), регистрируется как `CheckInterface` за `OptionalPackage`; без пакета — строка «sample: indexnowkit/verify is not installed» | core HTML не парсит |

### 5.2. Ротация ключа

`KeyGenerateRunner --force`: старое значение → `INDEXNOW_PREVIOUS_KEY=`; если переменная **уже непуста** — отказ с текстом
«INDEXNOW_PREVIOUS_KEY is still set from an earlier rotation; engines may still verify against it. Remove it first, or pass
--no-previous to drop it, or --yes to overwrite» (защита от двойной ротации). `operations.md` «Key rotation» под новое поведение.

### 5.3. Эскалация 403 между процессами

Без нового интерфейса: `Client::__construct(..., ?CacheInterface $failureCache = null, int $failureCacheTtl = 3600)`
(appended, тир Call). Счётчик `indexnow:403:{host}` с TTL; при наличии `increment()` у стора (Redis/Memcached через
адаптерный PSR-16) — атомарно, иначе `get/set` с оговоркой «приблизительно». Эскалация — пересечение порога: critical,
когда `count >= N` и флаг `indexnow:403:{host}:escalated` (TTL) не стоит; `reset()` только если счётчик ненулевой.
`Checker::probe()` строит `Client` без кэша — live-probe счётчик не трогает. Адаптеры передают тот же PSR-16, что для
debounce. Документация: критическая строка теперь срабатывает и в PHP-FPM.

### 5.4. `Submission\SubmissionStoreInterface`

```php
namespace IndexNowKit\Submission;

interface SubmissionStoreInterface
{
    /** одна запись на один Result (батч × endpoint); хранятся и skip-результаты */
    public function record(Result $result, \DateTimeImmutable $at): void;
    /** @return iterable<SubmissionRecord> новые первыми; фильтры по host, статусу */
    public function recent(int $limit = 100, ?string $host = null, ?ResultStatus $status = null): iterable;
    /** последняя по времени запись, чей `urls` содержит $url (любой статус) */
    public function lastFor(string $url): ?SubmissionRecord;
}
final readonly class SubmissionRecord { /** @param list<string> $urls */ public function __construct(public array $urls, public Result $result, public \DateTimeImmutable $at) {} }
final class NullSubmissionStore implements SubmissionStoreInterface {}
```

Подключение — не листенером (у него нет времени), а параметрами `Submitter::__construct(..., ?SubmissionStoreInterface
$store = null, ?ClockInterface $clock = null)` (appended; `SystemClock` по умолчанию). Адаптеры дают точку переопределения
с `NullSubmissionStore`. Таблица «состояние → записи»: один URL при `engines: ['api','yandex']` → две записи, `lastFor`
возвращает позднейшую; `Result::NO_ENGINE` (dry_run/skip) — тоже запись. Индекс url→record — забота реализации (`history`).

### 5.5. Канонизация URL

`Url\UrlNormalizerFactory::fromConfig(Config): UrlNormalizerInterface` в core **заменяет `new UrlNormalizer(...)` во всех
пяти местах core** (`Submitter:42`, `Client:44`, `IndexNowKit:94`, `Adapter/Services:113`, `Checker:177`) и в
`laravel/…ServiceProvider:160`; иначе опция молча не работала бы в plain PHP. `Url\CanonicalUrlNormalizer(UrlNormalizerInterface
$inner, …)` — декоратор: сначала `inner->normalize()`, затем канонизация; применяется в `Submitter` **до** дедупликации и
дебаунса. Блок `normalizer` в `Config::OPTIONS` точечно: `normalizer.strip_tracking_params` (bool, **default `true`** — члены
списка добавляются внешними источниками трафика, роутинг их не генерирует, ложных срабатываний нет по построению),
`normalizer.tracking_params` (list, дополняет), `normalizer.trailing_slash` (`keep` default | `add` | `strip` — может менять
идентичность страницы), `normalizer.sort_query` (bool, default false). `CanonicalUrlNormalizer::TRACKING_PARAMS` — растущая
константа (в `bc.md` рядом с правилом роста enum'ов): `utm_*`, `gclid`, `dclid`, `wbraid`, `gbraid`, `fbclid`, `msclkid`,
`yclid`, `ysclid`, `_openstat`, `etext`, `ttclid`, `twclid`, `igshid`, `mc_cid`, `mc_eid`, `mkt_tok`, `_hsenc`, `_hsmi`, `_ga`.

### 5.6. Модель правил: `Condition`

- `Attribute\Param\Condition { evaluate(object $subject): bool }`; `Equals` реализует только его; `ParamValue` — закрытое
  множество из четырёх (docblock синхронизирован). Тип `when` во **всех восьми** местах — `string|Condition|Closure|null`
  (`IndexNow.php:55`, `IndexNowDefaults.php:36`, `IndexNowUrl.php:34`, `UrlRule.php:27,130`, `RuleCompiler.php:155,172`,
  `ParamExtractor.php:138`, `ChangeClassifier.php:66`); `ObjectChangeHandler.php:204-206` — порядок веток `match` после
  разрыва наследования. `Equals` в `params` — ошибка типов и рантайм-ошибка `ParamExtractor` с текстом §2.2.
- Детекция удаления по переходу `true → false` для пользовательского `Condition`: опциональный `Attribute\Param\FieldCondition
  extends Condition { field(): string; heldFor(mixed $oldValue): bool }`; без него `ChangeClassifier` считает, что условие
  «держалось до», и удаление не детектируется — записано в `attribute-reference.md`. Тир Implement с оговоркой pre-1.0.
- `ExplainRunner` печатает значения: `when: status ("draft") -> true — a non-empty string is truthy; use new Equals('status',
  'published')`; `explain --json`. «Anti-patterns» (§3.3), `.phpstorm.meta.php` (§3.1).

### 5.7. Прочее волны E

- `indexnow:config --json` (console): эффективная конфигурация после дефолтов и env, ключ маскирован — артефакт для
  баг-репортов и ассистентов. `indexnow:status` — **не** в E (зависит от `history`/очередей), волна F.
- Laravel и Yii2 передают событийный диспетчер в `Submitter` (Laravel `Event::dispatch` мост → Telescope; Yii2
  `EVENT_RESULT`); Laravel `AboutCommand::add('IndexNow', …)`.
- Yii2: `-v/-vv/-vvv` наследуются в `ConsoleOutput` вместо собственного `--verbose`.
- Новые `Reason` для `verify` (enum закрыт, сторонний пакет case добавить не может):

  | Case | `isSkip()` | `retryable` |
  |---|---|---|
  | `Noindex`, `RobotsDisallowed`, `NonCanonical` | да | нет |
  | `Redirected` (политика `skip`) | да | нет |
  | `OriginError` | да | да |

- `SitemapRunner`: warning в рантайме, когда прогон без `--changed-since` отправил больше `batch.max_urls` URL.
- Релизы: `core@0.8.0` → `console@0.2.0` → `testing@0.1.x` → `sitemap@0.3.x` → `doctrine@0.5.x` → `symfony-bundle@0.9.0` →
  `laravel@0.10.0` → `yii2@0.8.0`. Changed: `Equals` в `params` не принимается; `strip_tracking_params` включён (ключи
  дебаунса меняются один раз); `check --strict` рекомендован в деплое.

## 6. Волна F: новые опциональные пакеты (сначала `verify`, потом `history`)

### 6.1. `indexnowkit/verify` 0.1.0

- `Verify\PageSignals` — парсер `<meta name="robots">`, `X-Robots-Tag`, `<link rel="canonical">`, robots.txt (~120 строк, без
  зависимостей) — живёт здесь, не в core.
- `Verify\VerifyingSubmitter implements SubmitterInterface` — декоратор с pre-flight GET (не HEAD) и политиками `redirect:
  skip|follow`, `non_canonical: skip|replace`, `origin_error: skip|send`; кэш robots.txt на хост; `delay` после коммита;
  `--no-verify` у `sitemap`. Конфиг-блок `verify.*` (dotted в `OPTIONS` пакета). Выключен по умолчанию.
- `Verify\Check\SampleCheck implements CheckInterface`: `check --sample=<url>` (повторяемый) и `--sample-class=<FQCN>`
  (через `SubjectLoaderInterface` из console); без явного источника — строка «no sample given»; результаты **не поднимают
  уровень отчёта выше warning** (прод из CI может быть недоступен); печатает статус, noindex, canonical, robots.
- Регистрация в адаптерах за `OptionalPackage`; строка в `check` без пакета.

### 6.2. `indexnowkit/history` 0.1.0

- `Psr16SubmissionStore` (кольцевой буфер N записей, дефолт 500), `PdoSubmissionStore` (таблица `indexnow_submissions`,
  индексы по url и at, `purge(olderThan)`); миграции — примеры для Doctrine Migrations, Laravel, Yii2.
- `History\Console\Definitions::history()` + `HistoryRunner`: `indexnow:history [--host] [--status] [--url] [--limit] [--json]`;
  `indexnow:status` (коллектор, debounce store, счётчик 403, последняя успешная отправка, очередь — где адаптер знает).
- Symfony: вкладка в профилере; строка в `check`: «history: 1 240 records, last 3 min ago».

## 7. Политики и критерии 1.0

- **PHP:** минимальная версия поднимается в первом миноре после выхода предыдущей из security-поддержки (`^8.2` → `^8.3`
  в первом миноре после 2026-12-31). **Подъём минимальной PHP — не нарушение BC**: Composer не предложит новый минор на
  старом PHP (записать в `bc.md`). Таблица `php/docs/compatibility.md` (пакет × PHP × фреймворк × EOL); Symfony 6.4 LTS
  до ноября 2027 совместим с этим правилом. **Сделано** (§16): таблица лежит в `packages/core/docs/compatibility.md`
  (а не в `php/docs/` — у монорепо нет своего docs/, сайт собирает только docs пакетов), в nav сайта «Supported versions».
- **Критерии 1.0 core** (`91-roadmap.md`): ноль `Symfony\Component\Console\`/`PHPUnit\` в `src/`; ноль лживых `suggest`;
  тир «may grow» исчез; `ParamExtractor::registerReader()` заменён инъекцией (**сделано** core 0.10.0, §16); `RuleAwareUrlResolverInterface` закрыт;
  `Adapter\Services`/`VerifyingStaging` прошли критерий формы с Yii3; **один полный минор (0.9) без breaking** после E;
  `SubmissionStoreInterface`, `Condition`, `Http\Response::headers`, `CheckItem::code`, `Client` failure-cache прожили минор;
  **ни одного `@deprecated` члена на теге 1.0** (`serve_key_file` удаляется в 1.0); `UPGRADE.md` 0.x → 1.0 один на семейство;
  идентификаторы conformance (C01–C22, A01–A21, H01–H06) заморожены как кросс-языковой контракт; docs-сайт и README-разделы
  для ассистентов есть; `check --json` валидируется схемой.
- **Trademark «IndexNow».** Решение пользователя (§10); предлагаемая фиксация в `00-overview.md`: описательное
  использование названия протокола в имени `indexnowkit`, дисклеймер стоит; план отхода — переименование vendor'а
  (PHP через `replace`; npm/PyPI/RubyGems/Maven — отдельные шаги, бренд уже занят на npm); **триггеры пересмотра**: до
  покупки домена и до PR в `recipes-contrib` (оба шага резко повышают видимость; переименование сегодня почти бесплатно).
- **Домен**: не блокер. **Спонсорство**: нет. **Flex-рецепт**: PR после трекшна и решения по trademark.
- **Dependabot vs Renovate**, **Codecov vs статический порог** — §10.

## 8. Риски

- **Порядок релиза D**: адаптеры требуют `console ^0.1`, `testing` в `require-dev`; регистрация пакетов на Packagist до
  первого пуша адаптеров; `packagist-wait-main` читает `require` + `require-dev` — учтёт.
- **Красный `check` на стейджинге** (0a): ломает пайплайны, где `check` стоял без `--strict` и окружение слало нарочно;
  выход — явный `dry_run: false` → warning; описано в CHANGELOG.
- **`strip_tracking_params` по умолчанию** (E): меняет ключи дебаунса один раз; URL с трекингом больше не уходят —
  это цель; список закрытый, роутинг такие параметры не генерирует.
- **`Condition`** — breaking только для `Equals` в `params` (баг); каскад из восьми мест перечислен.
- **Docs-сайт**: переписывание ссылок скриптом; `--strict` требует полного `nav` — генерируется.
- **AI-артефакты** живут в README — `ReadmeAiNotesTest` держит их синхронными с `Definitions`/`OPTIONS`.
- **Волна 0b длинная**: не гейт для D/E; DoD отдельный.

## 9. Definition of Done (проверяемые критерии)

0a: тест четырёх состояний стейджинга; `check` staging-конфига адаптера → exit 1; тест дебаунса; `INDEXNOW_PREVIOUS_KEY`;
`Engine` ×2; 12 текстов; `DebounceStoreCheck` в бандле; README-дефекты; `bin/packagist-check`; topics/LICENSE/Issues-off/
Dependabot/`composer audit`/advisories; `authors`; бейджи; SECURITY SLA; релизы §2.5.

0b: `grep -ril 'noindex\|robots.txt\|canonical' packages/*/docs` не пуст; prod checklist — первая ссылка раздела Operations в
трёх README; раздел «Notes for AI assistants» ×6 и `ReadmeAiNotesTest` зелёный; `ReadmeQuickstartTest` ×3 зелёный; Boost
guideline; `.phpstorm.meta.php`; `context7.json` + репозиторий submitted; сайт опубликован, `llms.txt`/`llms-full.txt`
отдаются, `mkdocs build --strict` и `lychee` зелёные; `bin/config-table` и тест; Yii2 `configuration.md`/`testing.md`/
`multi-domain.md` по разделам не уступают Laravel; три RU-документа; внешний человек прошёл quickstart Yii2 без ошибок.

D: гейт `grep -rl 'Symfony\\Component\\Console\|PHPUnit\\' packages/core/src packages/sitemap/src` пуст; `grep -rn class_exists
packages/{symfony-bundle,laravel,yii2}/src` не содержит `SitemapReader`; классов `SitemapSupport` нет; core без
`symfony/console` в `suggest`/`require-dev`, без `phpunit` в `suggest`; `testing`/`console` на Packagist, branch-alias есть;
все адаптеры зелёные на `highest`/`lowest`; ~106 файлов core.

E: `check --json` валидируется `docs/check.schema.json` в тесте; каждый `CheckItem` имеет `code` (тест: нет item без code);
`--strict`; `Content-Type`/`Cache-Control`/robots/`previous_key` под тестами с `FakeTransport`; `--force` с непустым
`PREVIOUS_KEY` отказывает; счётчик 403 в PSR-16 с TTL под тестом; `SubmissionStoreInterface` + `NullSubmissionStore` во всех
адаптерах; `UrlNormalizerFactory` — ноль `new UrlNormalizer(` вне фабрики (grep); `Condition` восемь мест; `explain`
значения + `--json`; `config --json`; PSR-14 в Laravel/Yii2; `about`; таблица `Reason`.

F: `verify` и `history` на Packagist, за `OptionalPackage`, строки в `check`, документы; `check --sample` работает только с
пакетом.

## 10. Решения пользователя

1. Лицензия `indexnowkit/spec`: MIT (дефолт) или CC-BY-4.0 текст + MIT код.
2. Trademark: принять формулировку §7 с триггерами пересмотра.
3. `Engine::Yandex`: `yandex.com` или `www.yandex.com` — после одного реального POST.
4. Домен `indexnowkit.dev`: позже (дефолт).
5. RU-перевод: три документа (дефолт) или вся документация.
6. `history`: сразу после `verify` (дефолт) или по спросу. `verify` — делается (спрос доказан аудитом).
7. Dependabot с `groups` (дефолт) или Renovate (GitHub App).
8. Codecov (живой процент, сторонний сервис) или статический бейдж порога (дефолт).
9. Спонсорство: не заводить (дефолт).

## 11. Уточнения по реализации (0a, 2026-09-05)

Отклонения от §2, принятые при реализации, с причинами:

1. **`dryRunExplicit` и `null`.** `fromArray()` считает `dry_run` явным только при ненулевом значении:
   `($data['dry_run'] ?? null) !== null`, а не `array_key_exists`. Причина: конфиг-файлы читают env-переменные,
   которых может не быть (`env('INDEXNOW_DRY_RUN')` в Laravel даёт `null`), и `array_key_exists` делал бы любой
   такой файл «явным». Следствия в адаптерах: узел `dry_run` дерева бандла без `defaultFalse()` (обработанное дерево
   не должно выдумывать явный `false`); `config/indexnow.php` Laravel читает `env('INDEXNOW_DRY_RUN')` без каста —
   опубликованный до 0.8.0 файл с `(bool) env(..., false)` даёт warning вместо error, пока не переопубликован
   (в CHANGELOG).
2. **`with()`** делает копию явной только при изменении `dryRun`; прочие изменения сохраняют флаг (копия с другими
   движками — не решение о dry_run). `withDryRun()` — как в §2.1. Конструктор: `true` по умолчанию (код — явное решение).
3. **Строка `environment`**: `ok` в production и в non-production, когда ничего не уходит (`dry_run`/`enabled: false`);
   `warning` в non-production, когда отправка идёт (оба состояния «error» и «explicit warning»). Ключ в тексте —
   `KeyValidator::mask()`; при `hosts` без `key` — «the keys of N host(s)».
4. **Фикстуры адаптеров** получили явный `dry_run: false`: их окружения (`test`/`testing`) не production, иначе каждый
   `check`-тест был бы красным. Staging-тест каждого адаптера снимает ключ через `dry_run: null`.
5. **`Engine::InternetArchive`** объявлен по meta.json, но хост `internetarchive.indexnow.org` на 2026-09-05 не
   резолвится (NXDOMAIN у авторитетных NS `indexnow.org`); документировано как «достижим через `api`». Значение
   `amazon` оставлено при id реестра `amazonbot`. `api.indexnow.org/searchengines.json` отдавал страницу недоступности
   Bing; снимок взят с `www.indexnow.org/searchengines.json` (спека 01). Яндекс: оба хоста отвечают 422 без редиректа —
   `yandex.com` остаётся (§10.3 закрыт).
6. **Тексты ошибок**: адреса §2.2 указывали на строки до правок; `Config.php:192/198/214` — это `resolver`,
   `debounce.key_prefix`, `retry`. `Checker` печатает первые 60 байт тела (управляющие символы схлопнуты, ключ маскируется).
7. **`DebounceStoreCheck` в бандле**: probe-ключ `indexnowkit_check` (двоеточие — зарезервированный PSR-6 символ,
   `Psr16Cache` бросает); closure для core-проверки — сервис `Closure::fromCallable` над `Check\CacheProbe`,
   регистрируется только когда `debounce.store` — пул. `ContainerShapeTest` перегенерирован.
8. **`bin/packagist-check`** вызывается в конце `bin/tag` информационно (`|| true`): в момент тега сплит и Packagist ещё
   не видят версию; строгий режим — `--strict` после `packagist-wait`.
9. **Psalm taint** для sitemap: `packages/sitemap/psalm.xml` (errorLevel 4), workflow `psalm-taint.yml` по понедельникам и
   вручную; Psalm ставится глобально в джобе, не в `require-dev`. Первый прогон — 0 находок.
10. **Спека 90** упоминаний спонсорства не содержала — удалять нечего.
11. **Бейдж coverage** только у core и sitemap (у остальных пакетов нет coverage-джоба и floor'а); порядок бейджей
    единый, RU-README тоже.
12. **SEO-тексты §3.3 без кода** сделаны в 0a: «≠ индексация», BWT/Я.Вебмастер, 410/404/301, Bing URL Submission API,
    полный sitemap-прогон и `lastmod` — в README; `operations.md`/`attribute-reference.md` — остаются в 0b.
13. **`bin/packagist-check` читает HTML-страницу пакета**, не `packages/<pkg>.json`: у JSON-API `s-maxage=43200`
    (12 часов CDN), после «Update» он ещё полдня отдаёт старую версию, а HTML обновляется сразу. §2.4 говорил про
    `packages/*.json` — заменено.

## 12. Уточнения по реализации (0b, 2026-09-05)

1. **Quickstart-фикстура** — `tests/Readme/Post.php` (класс `Post`), не `QuickstartModel.php`: README показывает
   `class Post`, а PSR-4 требует совпадения имени файла и класса. Рядом `tests/Readme/Category.php` — связанный объект
   правила `via: 'category'`, в README не показывается. Golden-сравнение — только EN README (RU несёт тот же код с
   русскими комментариями). Фикстура бандла — полная ORM-сущность (README вырос на ~20 строк, зато копируется целиком);
   php-cs-fixer форматирует промоутед-свойства с атрибутами на отдельных строках — README следует за файлом.
2. **`ReadmeAiNotesTest`** ×6 — через `Testing\ReadmeAssertions` ядра (тир Call): раздел есть в EN и RU, PHP-сниппет с
   `use IndexNowKit\…`, каждая команда `indexnow:…`/`indexnow/…` — из семейного списка, каждый ключ в обратных кавычках с
   точкой — из `Config::OPTIONS` + ключей пакета + документированных синонимов. Пример опечатки в тексте — без кавычек.
3. **`bin/config-table`** работает внутри `packages/symfony-bundle` (vendor с core/sitemap/бандлом); `LARAVEL_OPTIONS`/
   `YII_OPTIONS` читаются как текст из исходников соседей. Дефолты — из дерева бандла, где его нет — константы ядра;
   `dispatch`/`debounce.store` отмечены как исключения. Тест `ConfigTableTest` в бандле пропускается вне монорепо;
   CI-проверка — на ячейке bundle/8.3/highest.
4. **AI-разделы**: `AGENTS.md` лежит в `php/` (корень публикуемого монорепо). `context7.json` исключает `src`/`tests`:
   индексируются README и `docs/*.md`. Boost-guideline — формат `@verbatim <code-snippet>` из документации Boost.
5. **Docs-сайт**: `bin/docs-collect` — Python (доступен в Actions и локально), генерирует `docs-site/docs/`,
   `mkdocs.yml` из `mkdocs.template.yml` и `llms*.txt`; RU-страницы попадают на сайт без nav (strict-сборка требует
   цели ссылок). lychee — с `--accept 403,429` и исключениями example.com/packagist/endpoint'ов. Включение Pages и
   submit в Context7 — действия пользователя; `homepage` в composer.json переключается на сайт в ближайшем релизе.
6. **SEO-тексты §3.3** размещены: «≠ индексация», BWT/Вебмастер, удалённые страницы, «что не отправлять», мониторинг +
   Sentry — `operations.md`; антипаттерны — `attribute-reference.md`; hreflang/www-apex — `multi-domain.md` трёх
   адаптеров (у Yii2 документ новый). Код для warning полного sitemap-прогона — волна E, как в спеке.
7. **RU-перевод** — пять документов по дефолту §10.5; генерируемая таблица ключей остаётся в EN-файле, RU ссылается.
8. **Внешний проход quickstart Yii2** выполнен свежим агентом в `yii2-app-basic` (Docker, Packagist 0.6.0): три
   блокера и три ловушки исправлены в README EN/RU — установка сразу с PSR-18 клиентом; `'dry_run' => YII_ENV_DEV` в
   Install-сниппете; Yii2 не читает `.env` (export / phpdotenv); `urlManager` и компонент в обоих конфигах
   (`web.php`/`console.php` независимы в app-basic); имена классов в командах в кавычках (`'app\models\Post'` —
   shell съедает обратные слэши; то же в README бандла/Laravel); `namespace app\models`, колонки модели и
   иллюстративность `Category` — текстом перед golden-блоком. Pages включён (`gh api … /pages -f build_type=workflow`),
   сайт опубликован, `homepage` в composer.json и на GitHub → сайт.

## 13. Уточнения по реализации (D, 2026-09-05)

Отклонения от §4, принятые при реализации, с причинами:

1. **`ReadmeAssertions`** (появился в 0b, §4.1 его не перечисляет) переезжает в `testing` вместе с двумя другими
   ассерциями: `Testing\Conformance\ReadmeAssertions`. Причина — импортирует `PHPUnit\Framework\Assert`, как они.
   Тест README ядра (`ReadmeAiNotesTest`) переехал в `testing/tests` как `CoreReadmeAiNotesTest`: проверяет `../core`,
   вне монорепо пропускается — core не получает `require-dev: indexnowkit/testing` (бутстрап-цикл §4.1).
2. **`core/tests/Conformance/CoreConformanceTest.php` остаётся в core**: это собственный прогон C01–C22 ядра поверх
   `FakeTransport`, кит он никогда не использовал. В `testing/tests` добавлен `CoreConformanceKitTest` — кит против
   голого фасада `IndexNowKit::create()` (эталон адаптерного теста и покрытие пакета).
3. **Mock-server — две одинаковые копии**: опубликованная `testing/resources/mock-server/router.php` и приватная
   `core/tests/Support/mock-server/router.php` (тесты `Psr18Transport*` сплита core не могут ссылаться на соседний
   пакет). Синхронность держит `MockServerRouterTest` пакета `testing` (байтовое равенство в монорепо; тот же тест
   гоняет роутер под встроенным сервером по сценариям). `sitemap` хранит свой, другой роутер.
4. **Гейт «`Symfony\Component\Console\` отсутствует в `packages/sitemap/src`» не выполним без смены контракта**:
   `Sitemap\Console\SitemapRunner` — тело команды, печатает через `SymfonyStyle`, как раннеры `console`, а
   `ResultFormatterInterface::summary()` принимает `SymfonyStyle`. Импорт оставлен и объявлен: `sitemap` требует
   `indexnowkit/console ^0.1` **и** `symfony/console` явно (из `suggest` ушёл). Гейт для `packages/core/src` выполнен.
5. **`OptionalPackage::checkLevel()`** добавлен рядом с `check()`: compile-time контейнер бандла регистрирует
   `StaticCheck` определением с аргументами (уровень, строка), объект `StaticCheck` в него не положить. Уровень
   «блок настроен, но игнорируется» — warning (в 0.6 было ok); exit-код не меняется (`--strict` — волна E).
6. **Переопределение предиката в Laravel** — binding `IndexNowKitServiceProvider::SITEMAP_PACKAGE`
   (`'indexnowkit.sitemap_package'`, не `SitemapPackage::class`: у семейства будут ещё опциональные пакеты),
   фабрика `SitemapServices::package(?bool)`. Testbench: только `overrideApplicationBindings()` выполняется до
   `register()` провайдера; `defineEnvironment()` — после (провайдер уже выбрал ветку). `ConfigFactory::factory()/
   create()/build()` Laravel и Yii2 получили appended `?bool $sitemapInstalled = null`. В Yii2 свойство компонента
   `sitemapInstalled` соседствует с одноимённым методом-аксессором (оба публичны, как в §4.3).
7. **Бандл** тоже перешёл на `OptionalPackage` (гейт §9 D требует отсутствия `class_exists(SitemapReader)` во всех
   трёх адаптерах): `DependencyInjection\SitemapServices::package(?bool)`; константы `IndexNowKitLoader::SITEMAP_MISSING*`
   и `SitemapNotInstalledCommand::MESSAGE` удалены (тексты — из объекта; `DependencyInjection\*` — проводка, не API).
   Текст critical-строки Yii2 получил хвост `(run "php yii indexnow/check")` (единый через `loadOrDisabled()`).
8. **Состав core после D**: 106 файлов в `src/` (спека ожидала ~106; 105 после выноса плюс `Adapter\OptionalPackage`). `Submission\ResultSummary` — первый класс
   нового пространства `Submission\` (волна E добавит `SubmissionStoreInterface`).
9. **Версии и алиасы**: sitemap 0.3.0 требует `core ^0.7`, `console ^0.1`; адаптеры — `sitemap ^0.3` в
   `require-dev`/`suggest`, бандл — `doctrine ^0.5`; branch-alias подняты до релизных (0.7.x/0.1.x/0.1.x/0.3.x/0.5.x/
   0.8.x/0.9.x/0.7.x) до тега — `bin/link` разрешает соседей по алиасу.


## 14. Уточнения по реализации (E, 2026-09-06)

Отклонения от §5, принятые при реализации, с причинами:

1. **`--host` многократный без смены `CheckerInterface`.** `CheckerInterface::run(bool, ?string $onlyHost, ?string)` не
   тронут (tier «may grow», но расширение типа параметра ломает сторонние реализации без выигрыша): `CheckRunner`
   принимает `string|list<string>|null $host`, при нескольких хостах гоняет чекер по одному разу на хост и сливает
   отчёты — глобальные строки один раз, строки хостов по каждому. Адаптерные проверки при этом выполняются N раз
   (диагностика, N мал). Yii2: массив-свойство `host`, `--host=a,b` (Yii не поддерживает повтор опции).
2. **Схема JSON валидируется `justinrainbow/json-schema ^6.0`** в `require-dev` console (единственный тест); в
   `require` пакета зависимостей не добавилось. `items[].code` допускает `null` для проверок приложения без кода;
   при невалидной конфигурации `--json` печатает документ с одним item `config.invalid`.
3. **Коды `check`** (`core/docs/check-codes.md`): `next` в отчёт не попал — строка «Next: …» печатается `CheckRunner`
   после отчёта и в JSON не входит. `key_file.status` — ok для совпавшего файла, error при статусе ≠ 200; несовпавшее
   тело — отдельный `key_file.body`; транспортная ошибка и отсутствие клиента — `key_file.fetch`.
4. **`Content-Type` без заголовка при доступных заголовках — warning, не ok**: спека предписывала ok только для
   транспорта, который заголовков не отдаёт (`Response::$headers === []`, одна нейтральная строка); реально
   отсутствующий заголовок при отдаваемых остальных — реальный недостаток сервера. Проверки заголовков идут только
   после совпавшего ключевого файла; `Cache-Control` без заголовка — строки нет; `Age` > лимита — отдельный текст про
   CDN. `robots.txt` разбирается по группам `User-agent` (`*` и боты движков; Googlebot игнорируется), «длиннейшее
   правило побеждает, Allow при равенстве»; 404/ошибка robots — молчание. `previous_key` берётся из
   `hosts.<host>.previous_key ?? previous_key`, ключ маскируется.
5. **Ключи счётчика 403** — `<debounce.key_prefix>403.<host>` и `…_escalated`, не `indexnow:403:{host}`: двоеточие —
   зарезервированный PSR-6 символ, `Psr16Cache` бандла бросает (тот же урок, что §11.7). Эскалация в кэше — флаг рядом со
   счётчиком; при `increment()` свежий счётчик получает TTL отдельным `set`; исключение кэша логируется один раз
   (`warning`), процесс считает сам. Yii2 получил `Cache\Psr16Cache` над `yii\caching\CacheInterface` (у Yii нет
   PSR-16); в Laravel — `Cache::store()` (уже PSR-16, с `increment()`); при `debounce.store: memory|none` кэш не
   передаётся. `IndexNowKit::create()` получил `failureCache:` и `submissionStore:` (тринадцатый и четырнадцатый аргумент).
6. **`SubmissionStoreInterface`**: запись после листенеров и PSR-14, одно время `now()` на вызов `submit()`; dry-run
   результат несёт движок (`engine: api`), а не `NO_ENGINE`, — так `Client` строит `Result::skipped(...)` при dry_run
   с волны 0a; `NO_ENGINE` — у `disabled`/`debounced`/`no_key`/`invalid_url`. Документ — `core/docs/submission-store.md`.
7. **Канонизация**: `Checker::hostsToCheck()` тоже через фабрику (гейт grep). `trailing_slash: add` не трогает путь,
   последний сегмент которого содержит точку (файл), `strip` не трогает корень. `sort_query` — стабильная сортировка по
   имени (одинаковые имена сохраняют порядок). Значения параметров и их кодирование не меняются; сравнение имён —
   после `rawurldecode`, без учёта регистра. Дерево бандла получило узел `normalizer`, Laravel — блок в
   `config/indexnow.php`.
8. **`Condition`**: тип `when` — `string|Condition|Closure|null`, `ParamValue` (`Accessor`, `Value`, …) в `when` больше
   не принимается (TypeError; миграция `new Accessor('x')` → `'x'`). `Equals` в `params` — `ConfigurationException` из
   `ParamExtractor::extract()` с текстом, куда его положить. `.phpstorm.meta.php` не менялся: `when` не закрытое
   множество, подсказывать нечего. `explain` печатает значение каждого условия (`status ("draft") -> true — …`);
   `explain --json` — документ `class/id/event/config/rules[]/delivery[]/submits`.
9. **`indexnow:config`**: `Config::toArray()` в core (эффективная конфигурация в форме `fromArray()`, без маскировки);
   `ConfigRunner` маскирует `key`, `previous_key` и ключи `hosts`, добавляет секцию `adapter` (ключи сырой конфигурации,
   которых нет в `Config::OPTIONS`, как есть), `endpoints` и `core` (версия). Описание опции `--json` без фигурных
   скобок: парсер `$signature` Laravel считает `{` границей аргумента.
10. **PSR-14**: у Yii2 не было `EVENT_RESULT` (инвентарь ошибался) — добавлены `IndexNowComponent::EVENT_RESULT`,
    `Event\ResultEvent`, `Event\ResultDispatcher`; консольные сабмиттеры Yii2 берутся из
    `Services::submitterFactory()`, чтобы публиковать в тот же диспетчер и писать в тот же стор. Yii2 `-v/-vv/-vvv` —
    три булевых опции контроллера (`v`, `vv`, `vvv`; Yii не различает повторы одной буквы), плюс `SHELL_VERBOSITY`;
    `--verbose` удалён (Changed). Laravel: мост `Event\EventDispatcherBridge` под id `IndexNowKitServiceProvider::EVENTS`,
    секция `about` — версия core, enabled/dry_run, окружение, base_url, маскированный ключ, движки, dispatch, debounce.
11. **`Reason`**: добавлен `isRetryable()` (у enum не было понятия retryable; таблица в `bc.md` — по нему). Всего 17
    кейсов.
12. **Warning полного sitemap-прогона** — после сводки, при `--json` в stderr; порог — строго больше `batch.max_urls`.
13. **Тесты адаптеров**: `--host` теперь делает два GET на хост (ключевой файл и robots.txt); `explain` печатает
    `when: published (false) -> false`; фикстуры без блока `queue` — секция `adapter` показывает `router`/`sitemap`.

## 15. Уточнения по реализации (F, 2026-09-06)

Отклонения от §6 и от промпта волны, принятые при реализации, с причинами:

1. **`Retry\ForbiddenCounter::__construct(?CacheInterface $cache, string $keyPrefix, int $threshold, int $ttl = TTL, LoggerInterface $logger = NullLogger)`** —
   `threshold` раньше `ttl` (обязательный параметр не может идти после необязательного). Ключи
   `<prefix>403.<host>[_escalated]`. `Services::forbiddenCounter()` строит его из существующих узлов; сигнатура `Client`
   не менялась, тесты `Client` про 403 не тронуты.
2. **Core 0.9.0 кроме `ForbiddenCounter`** получил аддитивный `$extraHeaders` в `TransportFactory::lazy(Config, ?Closure,
   array $extraHeaders = [])` и `Psr18Transport::discover(?ClientInterface, ?float, array $extraHeaders = [])` —
   единственный способ дать User-Agent GET-ам verify без изменения `TransportInterface::get()`.
3. **`VerifyingSubmitter(SubmitterInterface $inner, TransportInterface $transport, VerifyConfig $config, KeyProviderInterface $keys, UrlNormalizerInterface $normalizer, LoggerInterface $logger = NullLogger, ?EventDispatcherInterface $events = null, ?SubmissionStoreInterface $store = null, ?RobotsCache $robots = null, ?ClockInterface $clock = null, bool $inWebRequest = false, ?Closure $sleep = null)`** —
   `KeyProviderInterface` вместо списка хостов, `RobotsCache` вместо `?CacheInterface`. Критерий «чужой хост»:
   `managedHosts()`, если перечислимы, иначе `keyFor()`.
4. **`PageSignals`** ~185 строк + `UrlReference` (RFC 3986 resolve), regex-парсер `<head>`, без `ext-dom`; `MAX_BYTES`
   262 144.
5. **`verify` не зависит от `console`**; `SampleCheck` — `CheckInterface` с замыканием-сэмплером класса из адаптера; опции
   `--sample`/`--sample-class` попадают в проверку через адаптерный holder `SampleOptions` + `VerifySampleCheck` (в каждом
   адаптере свой sampler: `EntitySampler`, `ModelSampler`, `RecordSampler`).
6. **`VerifyingSubmitterFactory`** (декоратор `SubmitterFactoryInterface`, `inner()` для sitemap без предпроверки); флаг
   sitemap — `no-verify` (`SitemapOptions(..., bool $noVerify)`, `SitemapRunner(..., ?SubmitterFactoryInterface $unverifiedSubmitters)`).
7. **`ConfigRunner::run(..., array $packages = [])`** — эффективные значения блоков (`toArray()`), а не `OPTIONS`.
8. **Политики verify**: robots 404 — «нет robots», без записи в лог; `origin_error: send` пишет warning; `non_canonical:
   replace` публикует skipped `NonCanonical` «submitted instead» для исходного URL и отправляет canonical; `redirect:
   follow` — 301/308 дают оба URL (исходный + цель), 302/303/307 — только исходный.
9. **`verify.dispatch`** — warning только при `dispatch: sync`. Кода **`verify.config` нет**: невалидный блок `verify`
   логируется `critical` в `VerifyConfig::loadOrDisabled()` и предпроверка выключена (в бандле блок отвергается деревом
   при компиляции); `check` печатает `verify.installed`, `verify.dispatch`, `verify.sample`.
10. **Yii2**: свойства компонента `verifyInstalled`, `verifyTransport`, `samples`, `historyInstalled`; методы `events()`,
    `historyPackage()/historyInstalled()/historyConfig()/historyEnabled()`.
11. **history**: `RecordCodec` (@internal), `Psr16SubmissionStore` — кольцевой буфер `history.index` + слоты
    `history.<n>`; `PdoSubmissionStore` — строка на URL, группировка по 26-символьному batch id; `--since` принимает
    `m/h/d/w`; `HistoryRunner(SubmissionStoreInterface, HistoryConfig, ?UrlNormalizerInterface, ?ClockInterface)`,
    `StatusRunner(Config, KeyProviderInterface, ForbiddenCounter, string $debounceStore, ?SubmissionStoreInterface, ?Closure $adapterFacts, ?ClockInterface)`.
12. **`HistoryConfig`** отвергает `pdo.dsn` вместе с `pdo.service` (одно правило на три адаптера; бандл дублирует его в
    дереве). `HistoryCheck`, `StatusRunner` и `HistoryRunner` считают `NullSubmissionStore` ядра «стором нет» — адаптеры
    передают свой стор как есть, пользовательский стор виден как `custom`.
13. **Маркер `OptionalPackage` для history — `HistoryConfig::class`**, не `HistoryStoreInterface`: предикат —
    `class_exists()`, который для интерфейса возвращает false.
14. **Подстановка стора в адаптерах**: бандл — `indexnowkit.submission_store` становится стором пакета (алиас
    `indexnowkit.history.store`), сервис приложения под тем же id по-прежнему побеждает (MergeExtensionConfigurationPass);
    Laravel — `extend()` binding'а `SubmissionStoreInterface`, который заменяет только `NullSubmissionStore`; Yii2 —
    узел `ServicesBuilder::submissionStore()` при `historyEnabled()` (свойство `submissionStore` компонента имеет
    приоритет). `pdo.service`: бандл — имя DBAL-соединения (`default` → `doctrine.dbal.default_connection`) или id
    сервиса (`getNativeConnection()` либо готовый `PDO`); Laravel — `DB::connection($name)->getPdo()`; Yii2 —
    `Connection::getMasterPdo()` компонента `db`. `psr16`: PSR-16-вид пула дебаунса (при `memory`/`none` — `cache.app`
    / стор кэша по умолчанию / компонент `cache`).
15. **`status`**: `$debounceStore` — `cache.app (ArrayAdapter)` в бандле, `cache (array)` в Laravel, `cache (ArrayCache)`
    в Yii2, `memory`/`none` как есть; `$adapterFacts` — бандл `transport` (сконфигурированный, либо «routed by
    framework.messenger.routing», либо «none: handled synchronously (set messenger.transport)») и `bus`; Laravel
    `connection` и `queue` (дефолты приложения, когда блок `queue` их не задаёт); Yii2 `component` и `class`.
16. **Заглушки без пакета**: бандл и Laravel — `HistoryNotInstalledCommand`/`StatusNotInstalledCommand` (exit 1, строка
    установки); Yii2 — действия `indexnow/history` и `indexnow/status` существуют всегда (как `indexnow/sitemap`),
    без пакета печатают строку установки и exit 1, их опции принимаются (`help` показывает действия — убрать их из
    списка Yii не позволяет без переопределения `actions()`).
17. **`--purge`** в трёх CLI: Symfony `VALUE_OPTIONAL` с дефолтом `false` (absent) → `null`/`true`/строка; Laravel
    `{--purge=}` + `hasParameterOption('--purge')`; Yii2 — свойство `purge` (`true` без значения, строка с `=`).
    `--host` у Yii2 переиспользует массивную опцию `check` и берёт первое значение; `--limit` без явного значения — 50.
18. **Профилер бандла**: `IndexNowDataCollector` — седьмой параметр `?SubmissionStoreInterface $history = null`,
    `lateCollect()` читает `recent(20)`; ошибка стора — одна строка панели. Laravel `about`: строка `History`
    (`pdo (indexnow_submissions), 1 240 records` / `psr16 (500 records kept), …` / `off (history.store)` /
    `custom (<class>)` / `…, store failed: <message>`).
19. **phpstan бандла** сканирует `ContainerConfigurator.php` (`scanFiles`): статическая рефлексия находит функции
    `service()`/`service_closure()` только в загруженном файле, и с ростом числа файлов порядок воркеров стал это ломать.

## 16. Уточнения по реализации (после F: пункты §7 без Yii3 и без тега 1.0; core 0.10.0)

Решение пользователя 2026-09-06: **1.0 не ставим**, пока нет полной уверенности в стабильности; долги §7 закрываются
обычными 0.x-релизами. Из шести пунктов §7 без Yii3 и без самого тега делаются два — они и сделаны:

1. **`ParamExtractor` — объект, а не статический реестр.** `new ParamExtractor(...$readers)` держит `SubjectReaderInterface`
   одного графа; `extract()/read()/resolve()/condition()` — методы экземпляра с теми же сигнатурами; `with(...)`,
   `fromReaders(iterable)` (для tagged iterator контейнера), `readers()`. `registerReader()/unregisterReader()` удалены
   (core 0.10.0, «Changed» с миграцией). Прокидка — только добавленными необязательными параметрами: `IndexNowKit::create(extractor:)`
   и конструктор (`public readonly ParamExtractor $extractor`), `AttributeUrlResolver` (конструктор, `fromConfig()`, `extractor()`),
   `ObjectChangeHandler`, `UrlRule::appliesTo(object, ?ParamExtractor)`, `ChangeClassifier::classify(..., ?ParamExtractor)`;
   null = чистый DSL (plain PHP и Doctrine ничего не замечают). Узел графа `Services::paramExtractor()` /
   `ServicesBuilder::paramExtractor()`. Адаптеры: Laravel — binding `ParamExtractor::class` (`extend()` + `with()` для своих
   ридеров), Symfony — сервис `indexnowkit.param_extractor` из `fromReaders(tagged_iterator('indexnowkit.subject_reader'))`,
   `SubjectReaderInterface` автоконфигурируется тегом, Yii2 — узел в `Wiring`, статический флаг `$readerRegistered` исчез.
   Console `ExplainRunner` читает через `$indexNow->extractor`; doctrine `IndexNowListener` передаёт его в свой
   `ObjectChangeHandler`.
2. **`FieldCondition` считается через экстрактор**: `condition()` читает `field()` ридерами и спрашивает `heldFor()`, поэтому
   `new Equals('status', 'published')` видит атрибут Eloquent/AR так же, как `params`. Иначе после ухода статики `Equals::evaluate()`
   на модели Eloquent ломался бы (интерфейс `Condition::evaluate(object)` — Implement-тир, менять сигнатуру нельзя).
   `Equals::evaluate()` сам по себе — чистый DSL; ядро его для `FieldCondition` не вызывает. Обычный `Condition` — `evaluate()`.
3. **`compatibility.md`** — `packages/core/docs/compatibility.md`: политика (минимальный PHP поднимается в первом миноре после
   выхода предыдущей версии из security-поддержки; это не BC-break — записано и в `bc.md`), матрица пакет × PHP × фреймворк
   × upstream EOL со ссылками, флейворы CI. В nav сайта — «Supported versions» (`bin/docs-collect`), ссылка в `php/README.md`.
4. **Релиз — выпущен 2026-09-06** (Packagist, GitHub releases, `packagist-check --strict` ×10, split-CI зелёный): core 0.10.0 (breaking), symfony-bundle 0.11.0 / laravel 0.12.0 / yii2 0.10.0 (новая точка расширения), console 0.3.1
   и doctrine 0.7.1 (код), testing 0.2.1 / sitemap 0.5.1 / verify 0.1.1 / history 0.1.1 (только `core ^0.10`). Branch-alias
   `dev-main` core → `0.10.x-dev` до тегов, иначе `composer.monorepo.json` соседей не резолвится.

Остаётся до 1.0 (по решению пользователя — без даты): критерий формы `Services`/`VerifyingStaging` с Yii3, `UPGRADE.md`,
ноль `@deprecated` (`serve_key_file` удаляется на 1.0), заморозка идентификаторов conformance C01–C22, A01–A21, H01–H06, S01–S08.

### 16.1. Волна G — стабилизация по аудиту 0.10 (2026-09-06)

Шесть линз (`docs/plans/audit-0.10.md`, отчёты в `docs/plans/audit-0.10/`). Закрыто всё, что не требовало решения
пользователя (~50 пунктов): секреты в `indexnow:config` (DSN, `key_location`, блоки пакетов через `ConfigRunner::maskedBlock()`,
`SECURITY.md` — два секрета), потеря/дубли URL (Doctrine `rollBack()` в `finally`, DBAL 3 `commit(): false`, ретраи Messenger/
Laravel только с `retryUrls`, чанкинг диспетчеров по `batch.max_urls`, `via`-обход с бюджетом depth × fan-out и посещёнными
объектами, `X-Robots-Tag` per-token, RFC 9309 выбор группы robots.txt, `RobotsCache` по origin, `TokenBucket` без двойного учёта,
`TransactionStaging::commit()` с catch), экстрактор фасада из резолвера (`Url\ParamExtractorAwareInterface`, `GuardedUrlResolver::inner()`),
Yii3-готовность графа (узлы `changes`/`clock`, `events` замыканием, `null` из замыкания для nullable-узлов, `ObserverHelper::forChanges()`),
`verify.time_budget`, лимит тела pre-flight 1 МиБ, общие `Verify\Check\{DispatchCheck,TransportCheck}`, PDO-стор транзакцией
и multi-row INSERT, атомарный `seq` ring-buffer, тесты/CI (doctrine C-конформанс, mysql/pgsql job, floor ×10, Laravel 13 lowest,
полнота AI-notes, реестр conformance-идов), документация (bc.md тиры: `RouteUrlResolverInterface`/`ResolverLocatorInterface` →
Implement, `ParamValue` — sealed; compatibility, README Yii2 «Verify»). ВЫПУЩЕНО 2026-09-07: core 0.11.0, console 0.4.0, testing 0.3.0,
sitemap 0.6.0, verify 0.2.0, history 0.2.0, doctrine 0.8.0, symfony-bundle 0.12.0, laravel 0.13.0, yii2 0.11.0. Первый прогон
нового CI дал два урока: `Schema::sql('pgsql')` ставил `BOOLEAN ... DEFAULT 0` (PostgreSQL отказывает, поймано mysql/pgsql
job'ом, исправлено в history 0.2.0), а полы coverage надо снимать с CI-джобы (pcov, setup-php 8.3), не с локального прогона.

Не сделано осознанно: L12 (уровень warning при превышении `verify.max_batch` — для `sitemap` это штатный путь, error шумел бы),
L4 (`RetryPolicy` берёт максимум `Retry-After` по хостам — смена семантики ретрая, отдельное решение), L9 (`collect()` без
`Result` для невалидного URL — семантика двух путей), W11 (`Config.php` 1026 строк — чистый рефакторинг, риск регрессии без выгоды
пользователю), адаптеры Laravel/Yii2 всё ещё строят фасад в хуке (узел `changes` есть; переключение — вместе с Yii3).
Решения пользователя из §6 аудита остаются открытыми.

### 16.2. Волна H — решения аудита 0.10 (2026-09-07)

Восемь `[решение]` из `docs/plans/audit-0.10.md` §6 приняты пользователем по рекомендации (все, кроме A10 — проводка
опциональных пакетов в сами пакеты, она первая задача Yii3-волны). Ломающие — одним минором core 0.12.0, чтобы Yii3 писался
на финальных сигнатурах:

- **A3(b)** `Attribute\Param\FieldCondition` больше не наследует `Condition`; `Equals::evaluate()` удалён (строил
  `new ParamExtractor()` внутри — на Eloquent отвечал неверно). `when: string|Condition|FieldCondition|Closure`.
  Вручную: `$extractor->condition($subject, $equals)`.
- **A4** экстрактор — обязательный параметр: `AttributeUrlResolver::__construct($reader, $extractor, ...)` / `fromConfig($config,
  $reader, $extractor, ...)`, `ObjectChangeHandler::__construct($rules, $resolver, $extractor, $logger)`,
  `ChangeClassifier::classify($rule, $subject, $extractor, $changedFields, $changeSet)`, `UrlRule::appliesTo($subject, $extractor)`;
  `ParamExtractor::plain()` для голого DSL. `IndexNowKit::create(extractor:)` и конструктор фасада — по-прежнему необязательны
  (фасад берёт экстрактор у резолвера).
- **A9** `IndexNowKit::create()` и конструктор без `resolver:` строят `AttributeUrlResolver::fromConfig()` (как `ServicesBuilder`),
  не `NullUrlResolver`; «ничего не резолвить» — явный `resolver: new NullUrlResolver()`.
- **A14** `IndexNowKit::submitEntities()`; `submitAll()` — `@deprecated`-алиас на один минор. Схема `submitX`/`submitXs` и словарь
  ядра записаны в `core/docs/adapters.md` («Names»).
- **A15** Yii2: `router.locales`, `router.locale_parameter`, `router.set_app_locale`; старые `router.languages`,
  `language_parameter`, `set_app_language` читаются с warning-депрекацией до следующего минора (`Wiring::ROUTER_RENAMED`).
- **A12** без `write()`: `ObjectChangeHandler::renamed(..., $previous)` зафиксирован в `bc.md` как контракт для адаптеров, чьи
  объекты не сбросить рефлексией; `SubjectReaderInterface` остаётся read-only.
- **W3** symfony/console в Yii2 остаётся (раннеры console-пакета, Laravel и Yii3 стоят на нём); дефект вывода мимо
  `Controller::stdout()` закрыт `Console\ControllerOutput` (наследник `Output`, `doWrite()` → `stdout()`, decorated по
  `isColorEnabled()`), инжектируемый `$output` не тронут.
- **W5** бандл: `|| ^8.0` на всех symfony/*, флейвор `ci:install:symfony8` и джоба PHP 8.4 в root- и split-CI; README и
  composer.json согласованы; `compatibility.md` обновлён.

Релиз: core 0.12.0, console 0.4.1, testing 0.3.1, sitemap 0.6.1, verify 0.2.1, history 0.2.1, doctrine 0.8.1,
symfony-bundle 0.13.0, laravel 0.13.1, yii2 0.12.0.
