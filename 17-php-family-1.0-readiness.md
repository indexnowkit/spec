# 17. Семейство PHP к 1.0: состав core, DX, UX, SEO, дистрибуция

Статус: спецификация v1, не реализовано. Основание: пять аудитов 2026-09-05 по линзам «состав core», «DX разработчика
и AI-ассистента», «UX владельца сайта», «SEO-корректность», «дистрибуция» (отчёты в scratchpad сессии, выводы
перенесены сюда). Исходное состояние: core 0.5.1, sitemap 0.1.1, doctrine 0.3.1, symfony-bundle 0.6.1, laravel 0.7.0,
yii2 0.5.0.

Цель: закрыть всё, что мешает назвать семейство готовым к 1.0 и удобным для трёх аудиторий — автора адаптера,
разработчика приложения (в том числе через AI-ассистента) и владельца сайта/SEO-специалиста, — по лучшим практикам
2026 года. Yii3 и Битрикс начинаются после этой спеки на её базе.

## 0. Сводка

| Линза | Сейчас | Цель | Главный разрыв |
|---|---|---|---|
| Состав core | 125 файлов, 10.2k строк; `symfony/console` и `phpunit` в `suggest`, хотя 12 файлов `src/` их импортируют | ~98 файлов, ноль `Symfony\`/`PHPUnit\` в `src/` | `Console\*` и `Testing\Conformance\*` — фичи, не ядро |
| DX человека | quickstart 7, ошибки 8, DSL 7, docs 8, testing 8 | 9 | три README-дефекта, перекос Yii2-доков, тихий отказ на ключах `router.locales`/`router.languages` |
| DX AI-ассистента | 3 | 9 | нет ни одного машиночитаемого артефакта (AGENTS.md, context7.json, Boost, llms.txt) |
| UX/эксплуатация | ключи 7, наблюдаемость 6, check 7, стейджинг **4**, CLI 8 | 9 | стейджинг с боевым ключом шлёт боевые запросы; `check` без `--json`/`--strict`; нет истории отправок |
| SEO | протокол 9, вред **5**, удаление 8, дедуп 8, честность 7 | 9 | ни слова про `noindex`/robots/canonical; «≠ индексация» не сказано; дефект дебаунса при нескольких движках |
| Дистрибуция | metadata 8, GitHub 5, discoverability 5, security 6 | 9 | topics/LICENSE/CoC/templates/Dependabot отсутствуют; витрина Packagist отстаёт |

Принципы (продолжают спеку 16):

1. **Фича — потребитель core.** Всё, что не нужно каждому адаптеру в веб-запросе, живёт отдельным пакетом с сохранением
   FQCN через PSR-4 (приём sitemap). Core не импортирует ни одного фреймворка и ни одного тестового фреймворка.
2. **Машиночитаемо по умолчанию.** Каждая проверка, конфиг и команда имеют структурный вывод/схему; документация
   индексируется и отдаётся ассистентам явно (AGENTS.md, context7.json, llms.txt).
3. **Честно.** IndexNow — уведомление, не индексация; Google не участвует; где смотреть результат — сказано.
4. **Безопасно по умолчанию.** Непроизводственное окружение не отправляет; `check` видит всё, что может навредить.

Волны:

| Волна | Релизы | Содержание | Риск |
|---|---|---|---|
| 0 | патчи README/доков адаптеров, без core | §2 гигиена, документация, AI-артефакты, docs-сайт | нулевой |
| D | core 0.6.0 + все адаптеры + testing 0.1.0 + console 0.1.0 | §3 состав: пакеты `testing`, `console`, `Adapter\OptionalPackage`; мелкие правки протокола | низкий (механика) |
| E | core 0.7.0 + все адаптеры | §4 эксплуатация и SEO: `check`, ротация, стейджинг, `SubmissionStoreInterface`, канонизация, DSL, тексты | средний |
| F | history 0.1.0, verify 0.1.0 | §5 новые опциональные пакеты | низкий, изолирован |
| — | core 0.8 → 1.0 | §6 политики, критерии 1.0, Yii3 как второй потребитель `Services` | — |

## 1. Целевая карта пакетов

```
L0  indexnowkit/core        протокол, модель правил, адаптерный кит, 4 тестовых двойника без PHPUnit
 ├─ L1 indexnowkit/console  раннеры CLI и Definitions            require: core, symfony/console ^6.4|^7|^8
 ├─ L1 indexnowkit/testing  conformance-киты и assertion-хелперы  require: core; suggest phpunit ^11
 ├─ L1 indexnowkit/sitemap  чтение sitemap                        require: core, ext-xmlreader; suggest console
 ├─ L1 indexnowkit/history  история отправок                      require: core; suggest console (волна F)
 └─ L1 indexnowkit/verify   pre-flight проверка URL               require: core (волна F)
L2  indexnowkit/doctrine    require: core (+ doctrine); dev: testing
    indexnowkit/{symfony-bundle,laravel,yii2,yii3,bitrix}  require: core, console; suggest: sitemap, history, verify; dev: testing
```

Вердикты по составу core (детали и цифры — в отчёте линзы; здесь решения):

| Кандидат | Строк | Решение | Основание |
|---|---:|---|---|
| `Console\*` кроме `SubmitterFactory*`, `ResultSummary` | ~1075 | **вынести** → `indexnowkit/console`, FQCN сохраняются | 8 файлов импортируют `symfony/console`, который в `suggest`; doctrine — единственный адаптер без CLI |
| `Console\SubmitterFactory`, `SubmitterFactoryInterface` | 71 | **переместить** → `Adapter\` | зовутся из `Adapter\Services`; слой 2 не должен знать про CLI |
| `Console\ResultSummary` | 64 | **переместить** → `IndexNowKit\ResultSummary` | общий для любых многобатчевых прогонов |
| `Testing\Conformance\*`, `KeyFileAssertions`, `CheckOutputAssertions` | 760 | **вынести** → `indexnowkit/testing` под префиксом `IndexNowKit\Testing\Conformance\` (хелперы переименовываются в этот namespace) | расширяют PHPUnit из `suggest`; ноль потребителей в `src`; `OrmConformanceTestCase` — 453 строки в прод-vendor |
| `Testing\{FakeTransport, ArrayLogger, FrozenClock, RecordingDispatcher}` | 212 | оставить | реализации интерфейсов core без PHPUnit; спека 10 обещает их в основном пакете |
| `Adapter\Services`/`ServicesBuilder` | 528 | оставить до Yii3; если Yii3 не станет вторым потребителем — вернуть в yii2 к 0.8 | один потребитель |
| `Check\*`, `Transaction\*`, `Key\*`, `Retry\*`, `Url\Punycode`, `Attribute\Param\*`, `Hook\*` | — | оставить | ноль внешних зависимостей, реальные потребители или протокольный шов |
| `Adapter\OptionalPackage` | +40 | **добавить** в core | три копии `SitemapSupport`/предиката в адаптерах |
| `SitemapConfig::loadOrDisabled()` | +25 | **добавить** в sitemap | три копии «невалидный блок → critical → disabled» |

Зависимости core остаются как есть (`php-http/discovery`, `psr/*` обязательны). Из `suggest` уходят `symfony/console` и
`phpunit/phpunit`. Не дробить: `Check`, `Key`, `Transaction`, `Attribute\Param`, по слоям «protocol/rules» — контрпримеры
в отчёте линзы, решение окончательное.

## 2. Волна 0: гигиена, документация, AI-артефакты (без изменения кода core)

Каждый пункт — отдельный коммит или группа коммитов; адаптеры получают патч-релизы только ради README/AGENTS.md
(файлы уходят в сплиты и на Packagist-страницы).

### 2.1. Packagist и GitHub

- **Витрина Packagist.** `repo.packagist.org/p2` актуален (Composer ставит верные версии), а `packagist.org/packages/*.json`
  и веб-страницы core/doctrine/symfony-bundle отстают на 2–4 релиза. Действие пользователя: «Update» на трёх страницах.
  Инструмент: `php/bin/packagist-check <pkg>` — сравнивает последний тег сплита, `p2` и `packages/*.json`, печатает
  расхождение; вызывается в конце `bin/tag` (warning, не ошибка) и руками.
- **GitHub topics** для `php-sitemap` (`indexnow sitemap seo php`), `php-laravel` (`indexnow laravel eloquent seo php`),
  `php-yii2` (`indexnow yii2 activerecord seo php`); `homepage` у всех сплитов = монорепо (пока нет сайта — §2.4).
- **`indexnowkit/spec`: LICENSE.** Спека — открытый эталон для портов: CC-BY-4.0 для текста (или MIT, если пользователь
  предпочтёт единообразие). Решение пользователя, дефолт CC-BY-4.0.
- **Read-only сплиты:** выключить Issues, Wiki, Projects на шести репо (`gh repo edit --enable-issues=false …`);
  CONTRIBUTING уже отправляет в монорепо.
- **`indexnowkit/.github`:** профиль организации (README: что это, таблица пакетов, ссылка на сайт),
  `CODE_OF_CONDUCT.md` (Contributor Covenant 2.1), `SECURITY.md` по умолчанию, `ISSUE_TEMPLATE/` (bug: пакет, версия,
  фреймворк, вывод `indexnow:check --json`, репродукция; feature), `PULL_REQUEST_TEMPLATE.md` (чеклист из CONTRIBUTING),
  `FUNDING.yml` — не создаётся (решение: спонсорства нет; убрать упоминание из спеки 90).
- **Монорепо `php/`:** `.github/dependabot.yml` (composer для шести пакетов, github-actions), `.github/workflows/codeql.yml`
  (PHP, публичный репо — бесплатно), `CODE_OF_CONDUCT.md` (ссылка на org).
- **composer.json:** `authors` в doctrine/symfony-bundle/laravel (как в core); `config.allow-plugins: {"php-http/discovery": false}`
  во всех адаптерах; keywords `psr-18`, `psr-3` у адаптеров.
- **Бейджи:** coverage (шаг CI генерирует `img.shields.io/badge/coverage-NN%25-…` в `php/README.md` и README core/sitemap
  через коммит бота? — нет: статический бейдж обновляется скриптом `bin/coverage-floor --badge` в релизном коммите),
  `phpstan level 9` (статический), license — во всех шести README, единый порядок.
- **SECURITY.md:** одна фраза SLA во всех семи файлах: «acknowledged within 5 business days; fix or mitigation plan
  within 30 days».

### 2.2. README-дефекты (ведут AI и человека к ошибке)

| Файл | Дефект | Правка |
|---|---|---|
| `laravel/README.md:10`, `README.ru.md`, `CONTRIBUTING.md:12` | бейдж и текст «Laravel 11» при `^12 \|\| ^13` | «12 \| 13»; CONTRIBUTING «Laravel 12 and 13» |
| `laravel/README.md:86-92`, `README.ru.md:90` | сниппет без `use IndexNowKit\Attribute\{IndexNow, IndexNowDefaults, RuleSet}` | добавить импорты |
| `yii2/README.md:22-41` | команды до конфигурации компонента (контроллер регистрируется в `bootstrap()`) | блоки местами; комментарий |
| `laravel/README.md` рядом с фасадом | два класса с коротким именем `IndexNowKit` (ядро и фасад) | абзац «какой импорт когда»; алиас-методы `submitEntity`/`submitEntities` в `IndexNowManager` и `IndexNowComponent` (делегаты на `submitModel`/`submitRecord`) |
| `core/README.md:211`, `symfony-bundle/docs/troubleshooting.md:47` | `serve_key_file` в пользовательских доках | убрать, оставить в CHANGELOG/bc |
| `sitemap/README.md` | нет Google-абзаца в пакете с именем «sitemap» | абзац из core README + строка про `lastmod = now()` |
| все README «Who gets notified» | нет Internet Archive и Amazon (endpoint'ы опубликованы) | список из `searchengines.json`; код — волна D |

### 2.3. AI-артефакты

- **`AGENTS.md`** в корне монорепо (как работать с репо: bin/*, коммиты, где спека) и в каждом пакете (уходит в сплит):
  имя на Packagist и минимальный `composer require`; сниппет «разметить сущность» с полными `use`; сниппет «отправить
  URL»; полный список валидных ключей конфига (генерируется из `Config::OPTIONS` + `<Adapter>::OPTIONS` скриптом
  `bin/agents-config-table`, чтобы не разъезжаться); команды с точными именами; **ловушки**: `dispatch: auto` есть в
  Symfony/Yii2 и нет в Laravel; `router.locales` (Laravel) vs `router.languages` (Yii2); `url:` — имя аксессора, `urls:` —
  литералы; строка в `when` truthy — нужен `Equals`; `submitEntity`/`submitModel`/`submitRecord`; фасад vs ядро в Laravel.
  Тест `AgentsFileTest` в каждом пакете: файл существует, все упомянутые команды есть в `Definitions`, все ключи — в
  `OPTIONS`.
- **`context7.json`** в корне монорепо: `folders: ["php/packages/*/docs", "php/packages/*/README.md", "php/packages/*/AGENTS.md"]`,
  `excludeFolders: ["**/vendor", "**/tests", "docs/plans", "docs/spec"]`, `rules` = те же ловушки.
- **Laravel Boost:** `laravel/resources/boost/guidelines/core.blade.php` (30–50 строк) — официальная точка расширения
  Boost для пакетов; содержимое = AGENTS.md пакета в формате Boost.
- **`.phpstorm.meta.php`** в core: `expectedArguments` для `IndexNow::__construct` `locales` (`'current'`, `'all'`) и
  `events` (`'created'`, `'updated'`, `'deleted'`).
- **`llms.txt` / `llms-full.txt`** — генерируются docs-сайтом (§2.4).
- **Doc-тесты:** `ReadmeQuickstartTest` в каждом адаптере прогоняет сниппеты README (разметка + первая отправка) через
  `FakeTransport`; README становится контрактом, дефекты §2.2 не повторяются. Механика: сниппеты помечаются
  `<!-- test: quickstart-model -->` … `<!-- /test -->`, тест извлекает их и `eval`-ит в песочнице теста.

### 2.4. Docs-сайт

MkDocs Material из монорепо на GitHub Pages (`indexnowkit.github.io/php`), workflow `docs.yml`: собирает
`php/packages/*/README.md` + `php/packages/*/docs/*.md` (без правок файлов: `mkdocs-monorepo-plugin` или копирование
скриптом), навигация: Getting started (Symfony / Laravel / Yii2 / plain PHP) → Attribute reference → Configuration
(с таблицей соответствия ключей §2.7) → Operations (prod checklist первым) → Testing → Adapters (для авторов) →
Packages. Поиск из коробки. Генерация `llms.txt` (оглавление со ссылками) и `llms-full.txt` (конкатенация) в корень
сайта. `homepage` в composer.json и на GitHub → сайт. Домен `indexnowkit.dev` — решение пользователя, не блокер:
Pages переезжает на домен одной настройкой.

### 2.5. SEO-честность и SEO-рекомендации в документации

- **«IndexNow ≠ индексация».** Абзац в шести README после «Who gets notified» и в `operations.md`: 200/202 = «уведомление
  принято»; решение о краулинге и индексации принимает движок (качество, robots.txt, `noindex`, canonical, лимиты);
  библиотека знает, дошло ли уведомление, и не знает, попадёт ли страница в индекс.
- **Где смотреть результат:** Bing Webmaster Tools → IndexNow / IndexNow Insights (категории «robots disallowed»,
  «content quality», «deadlinks»), Яндекс.Вебмастер → Индексирование → Переобход страниц. Метрика успеха — доля
  отправленных URL в индексе за N дней, читается только там. Раздел в `operations.md`, ссылка из README.
- **Что сайт должен отдавать на удалённом URL** (`operations.md`, новый раздел; три строки в README у «Deletion
  semantics»): 410 для удалённых навсегда, 301 + отправка обоих URL для замен, 404 для временно снятых; soft-404 и
  «мягкое удаление с редиректом на главную» — вред.
- **Чего слать нельзя** (`operations.md` + `attribute-reference.md` «Anti-patterns»): `noindex`/`X-Robots-Tag`,
  закрытые robots.txt, неканонические URL (`utm_*`, параметры, пагинация, фасеты), 3xx/4xx/5xx, черновики. Что уже
  защищает библиотека (`when`, `strict_hosts`, нормализатор) и что даст волна E/F (`CanonicalUrlNormalizer`,
  выборочная проверка в `check`, пакет `verify`).
- **Sitemap:** в README пакета и `docs/sitemap.md` адаптеров — полный прогон без `--changed-since` разовый (миграция,
  первый запуск); по расписанию только инкрементально (Bing: «streaming URLs as they change», batch mode); описание
  `--force` дополнить «снимает дебаунс — не для регулярных заданий»; строка про недостоверный `lastmod`.
- **Bing Webmaster URL Submission API и Google Indexing API — другое** (два предложения в README core): другой ключ,
  суточная квота 10 000/день против 10 000/запрос у IndexNow; Google — только `JobPosting`/`BroadcastEvent`.
- **hreflang-кластер:** пример с `via:` для переотправки языковых версий (`multi-domain.md`).
- **`batch.max_urls = 10000`** — потолок протокола, не цель (`configuration.md`).

### 2.6. Эксплуатационная документация

- **Prod checklist** — первый раздел `core/docs/operations.md`, ссылки из трёх README: ключ и base_url заданы,
  `check --strict` зелёный; `strict_hosts: true` при нескольких хостах; `debounce.store` — общий кэш; dispatch — очередь с
  воркером под мониторингом; стейджинг: `INDEXNOW_DRY_RUN=1` или `INDEXNOW_ENABLED=0` и `key_file.enabled: false`;
  алерты на «consecutive failures», «collected URL(s) discarded», «invalid configuration»; `check --strict` в деплое;
  `key_file.cache_max_age <= 300` и CDN его не переопределяет; после ротации `previous_key` снят через сутки.
- **Сценарий «отправились URL стейджинга»** в troubleshooting трёх адаптеров: как заметить, как остановить, что делать.
- **Сценарий «дубли / слишком часто»** (`debounce.store: memory` при нескольких воркерах) — там же.
- **Эскалация 403 per-process:** в `operations.md` оговорить, что счётчик живёт в процессе (PHP-FPM не досчитает);
  код — волна E (счётчик в общем кэше).
- **Ключ не уходит в query string** — одна фраза в SECURITY.md/operations.
- **Готовые правила мониторинга:** четыре правила Prometheus/Loki-стиля и фильтр Sentry в `operations.md`.
- **Yii2-паритет:** `configuration.md` (полная таблица ключей с дефолтами, как у Laravel), `testing.md` (транзакционная
  семантика verify-on-commit), новый `multi-domain.md`, разделы 422/429/дубли в troubleshooting.
- **www/apex** — абзац в `multi-domain.md` каждого адаптера.
- **RU-перевод** трёх документов: `core/docs/attribute-reference.md`, `core/docs/configuration.md`,
  `*/docs/troubleshooting.md` (`.ru.md` рядом). Решение пользователя по объёму; дефолт — эти три.

### 2.7. Таблица соответствия ключей адаптеров

В `core/docs/configuration.md` (и на сайте): одно понятие — три ключа (локали: `framework.enabled_locales` /
`router.locales` / `router.languages`; параметр локали; очередь: `messenger.transport` / `queue.connection` /
`queue.component`; хук ORM: `doctrine.*` / `eloquent.enabled` / `active_record.enabled`; `dispatch: auto` — где есть).
Код: Yii2 принимает `router.locales`, `router.locale_parameter`, `router.set_app_locale` как синонимы своих
`languages`-ключей (волна D, вместе с релизом yii2); старые остаются.

## 3. Волна D: состав core (core 0.6.0)

### 3.1. Пакет `indexnowkit/testing` 0.1.0

- `packages/testing`, autoload `IndexNowKit\Testing\Conformance\` → `src/`. Переезжают `CoreConformanceTestCase`,
  `OrmConformanceTestCase` (FQCN без изменений) и `KeyFileAssertions`, `CheckOutputAssertions` →
  `IndexNowKit\Testing\Conformance\{KeyFileAssertions, CheckOutputAssertions}` (переименование — breaking, миграция:
  один `use`). Mock-server роутер core (`tests/Support/mock-server`) копируется в `testing/resources/mock-server/` и
  документируется как «клонировать не нужно: `vendor/indexnowkit/testing/resources/mock-server/router.php`».
- `composer.json`: `require: php ^8.2, indexnowkit/core ^0.6`; `require-dev`/`suggest: phpunit/phpunit ^11`; в `src/`
  разрешён импорт PHPUnit (это его назначение).
- Адаптеры: `require-dev: indexnowkit/testing ^0.1`; `core/composer.json` без `phpunit` в `suggest`; core/docs/testing.md
  и adapters.md ссылаются на пакет.

### 3.2. Пакет `indexnowkit/console` 0.1.0

- `packages/console`, autoload `IndexNowKit\Console\` → `src/`. Переезжают 16 файлов: раннеры, `ResultRenderer`,
  `ResultFormatterInterface`, `CommandDefinition`, `ArgumentDefinition`, `OptionDefinition`, `Definitions`, `Vocabulary`,
  `ExitCode`, `ClassNameResolver`, `SubjectLoaderInterface`, `SubmitSubjectsOptions`. FQCN сохраняются.
- До переноса в core: `Console\SubmitterFactory`, `SubmitterFactoryInterface` → `Adapter\SubmitterFactory`,
  `Adapter\SubmitterFactoryInterface`; `Console\ResultSummary` → `IndexNowKit\ResultSummary` (breaking, миграция —
  `use`). `Adapter\Services::submitterFactory()` возвращает `Adapter\SubmitterFactoryInterface`.
- `composer.json`: `require: php ^8.2, indexnowkit/core ^0.6, symfony/console ^6.4 || ^7.0 || ^8.0`.
- Адаптеры symfony-bundle, laravel, yii2 и sitemap: `require: indexnowkit/console ^0.1`; doctrine не меняется (кроме
  `core ^0.6`). core: `symfony/console` уходит из `suggest` и `require-dev`; `bc.md` тиры для `Console\*` переезжают в
  `console/docs/bc.md`.
- Гейт: `grep -rl 'Symfony\\\|PHPUnit\\' packages/core/src` пуст.

### 3.3. `Adapter\OptionalPackage` (core) и `SitemapConfig::loadOrDisabled()` (sitemap 0.2.0)

```php
namespace IndexNowKit\Adapter;

final class OptionalPackage
{
    /** @param class-string $marker класс, существование которого означает «пакет установлен»; тест переопределяет через $installed */
    public function __construct(public readonly string $package, public readonly string $marker, public readonly string $feature) {}
    /** @internal для тестов */ public static array $installed = [];   // package => bool
    public function installed(): bool;
    public function notInstalledMessage(): string;                     // 'indexnowkit/sitemap is not installed: composer require indexnowkit/sitemap'
    /** @param array<string,mixed> $block @param array<string,mixed> $defaults */
    public function checkLine(array $block, array $defaults): string; // 'sitemap: not installed (...)' | '... the sitemap block in the configuration is ignored (...)'
    public function check(array $block, array $defaults): Check\StaticCheck;
}
```

`SitemapConfig::loadOrDisabled(array $block, LoggerInterface $logger, string $checkCommand): SitemapConfig` — «невалидный
блок → critical `indexnow: invalid sitemap configuration, the sitemap command is disabled until it is fixed: {error}
(run "{check}")` → `disabled()`». Три `SitemapSupport`/`SitemapServices` адаптеров сводятся к этим двум вызовам.

### 3.4. Мелкие правки core, едущие в 0.6.0

- **Дефект дебаунса при нескольких движках** (`Submitter.php:75`): маркировать только URL, у которых все endpoint'ы
  ответили успешно — `array_diff(urlsWhere(isSuccess), Result::retryableUrls($results))`; тест: `engines: ['api','yandex']`,
  200 + 503 → URL не в дебаунсе, ретрай доходит до второго движка.
- `Engine::InternetArchive = 'internetarchive'` (`https://internetarchive.indexnow.org/indexnow`), `Engine::Amazon =
  'amazon'` (`https://indexnow.amazonbot.amazon/indexnow`); один реальный запрос-проверка `yandex.com` vs
  `www.yandex.com` (редирект на POST теряет тело у части клиентов) — по результату поправить `Engine::Yandex`;
  `docs/spec/01-protocol.md:8-22` обновить.
- `Config::fromEnv()` читает `INDEXNOW_PREVIOUS_KEY`; таблица env в `configuration.md`.
- `Retry\RetryingSubmitter` docblock: «путь для sync/CLI; очереди используют `WorkerOutcome`».
- Yii2 синонимы ключей `router.locales*` (§2.7).

### 3.5. Релизы волны D

`core@0.6.0` → `testing@0.1.0` → `console@0.1.0` → `sitemap@0.2.0` → `doctrine@0.4.0` → `symfony-bundle@0.7.0` →
`laravel@0.8.0` → `yii2@0.6.0`. Новые репо `php-testing`, `php-console` (deploy-keys, секреты `SPLIT_SSH_KEY_TESTING`,
`SPLIT_SSH_KEY_CONSOLE`, Packagist-регистрация); `split.yml` стадии: core → {testing, console, sitemap} → адаптеры;
`packagist-wait-main` учитывает новые зависимости автоматически (читает composer.json). Матрица `ci.yml`, `php/README.md`,
`tools-push-splits.sh`, `bin/release-notes` карта.

## 4. Волна E: эксплуатация и SEO (core 0.7.0)

### 4.1. `indexnow:check` как healthcheck и SEO-проверка

| Изменение | Где | Суть |
|---|---|---|
| `--json` | `console` `Definitions::check()`, `CheckRunner` | `{"status":"ok\|warning\|error","environment":"prod","items":[{"level","message","host"?}],"hosts":{...}}`; без ANSI |
| `--strict` | там же | предупреждения дают exit 1; для CI/деплоя |
| `--host` многократный | там же | список хостов |
| Строка окружения всегда | `Check\Checker` | `environment: staging (not in production_environments: prod, production)` — ok в проде, warning вне |
| **Стейджинг с боевым ключом** | `Checker` после `dry_run` | `!isProduction && !dryRun && enabled` → **error**: «environment "staging" is not production but dry_run is off: changes WILL be sent to search engines under key ab12****. Set INDEXNOW_DRY_RUN=1 or INDEXNOW_ENABLED=0 outside production.» Уровень error (не warning): необратимые последствия |
| `Content-Type` ключевого файла | `Http\Response` получает `headers: array<string,string>` (lowercase keys) + `contentType()`; `Psr18Transport::get()` заполняет | `text/plain` обязателен; иначе error с фактическим типом |
| Фактический `Cache-Control`/`Age` | `Checker` | warning «CDN держит файл N с, ротация займёт N с», если `max-age` > `key_file.cache_max_age` |
| Редиректы на ключевом файле при пользовательском `http.client` | `Checker` | `check` строит отдельный транспорт без редиректов через `TransportFactory::lazy` с `followRedirects: false` (новый параметр `Psr18Transport::discover`), либо предупреждает, что клиент пользователя следует редиректам |
| robots.txt | `Checker` | GET `/robots.txt`, `Disallow`, совпадающий с путём ключевого файла → warning |
| `previous_key` | `Checker::checkHost` | если задан: отдаётся ли `/{previous}.txt` (200 → ok «rotation window open; remove previous_key after check --live is green»), иначе warning |
| Тело не совпало | текст | «got %d bytes starting with "%s"; a 200 answer with HTML usually means a catch-all route matched first» |
| **Выборочная SEO-проверка URL** | `Checker` `--sample[=N]` (default 3) | берёт N URL, которые библиотека реально отправила бы (через `SubjectLoader` первого класса с правилами, `explain`), делает GET: статус, `<meta name=robots noindex>`, `X-Robots-Tag`, `<link rel=canonical>` ≠ URL, robots.txt; печатает по строке; выключено без `--sample`, чтобы `check` не ходил по сайту молча |
| Следующий шаг | `CheckRunner` финал | после `IndexNow is ready.` — «Next: annotate a class with #[IndexNow(...)] or send one URL: <команда фреймворка> indexnow:submit https://…» (команда из `Vocabulary::checkCommand`-подобного поля) |
| `DebounceStoreCheck` в бандле | symfony-bundle | зарегистрировать с probe над `Psr16Cache` (паритет с Laravel/Yii2) |

### 4.2. Ротация ключа

- `KeyGenerateRunner --force`: старое значение пишется в `INDEXNOW_PREVIOUS_KEY=` (создать/заменить строку), печатается
  напоминание снять через сутки; `--no-previous` отключает.
- `operations.md` «Key rotation» переписывается под новое поведение; `check` показывает окно ротации (§4.1).

### 4.3. Эскалация 403 между процессами

`Client` считает подряд идущие 403 на хост в `DebounceStoreInterface`-подобном общем хранилище: новый интерфейс
`Key\FailureCounterInterface { increment(string $host): int; reset(string $host): void }` с `MemoryFailureCounter`
(как сейчас) и `Psr16FailureCounter`; адаптеры дают PSR-16 реализацию над тем же кэшем, что debounce. `Config`:
без новых опций (`forbidden_escalation` уже есть). Документация: критическая строка теперь срабатывает и в PHP-FPM.

### 4.4. `Submission\SubmissionStoreInterface`

```php
namespace IndexNowKit\Submission;

interface SubmissionStoreInterface
{
    public function record(Result $result, \DateTimeImmutable $at): void;
    /** @return iterable<SubmissionRecord> новые первыми */
    public function recent(int $limit = 100, ?string $host = null, ?ResultStatus $status = null): iterable;
    public function lastFor(string $url): ?SubmissionRecord;
}
final readonly class SubmissionRecord { public function __construct(public string $url, public Result $result, public \DateTimeImmutable $at) {} }
final class NullSubmissionStore implements SubmissionStoreInterface {}
```

Подключение через существующий `Submitter::addListener()` — конвейер не меняется. Адаптеры получают точку
переопределения (сервис/binding/свойство) с `NullSubmissionStore` по умолчанию. Реализации — пакет `history` (§5.1).
Контракт фиксируется до 1.0.

### 4.5. Канонизация URL

`Url\CanonicalUrlNormalizer implements UrlNormalizerInterface` — декоратор: срезает трекинг-параметры
(`utm_*`, `gclid`, `fbclid`, `yclid`, `msclkid`, `_ga`, `mc_cid`, `mc_eid`; список расширяемый), политика trailing slash
(`keep|add|strip`), сортировка query по ключу (опция). Конфиг: блок `normalizer` в `Config::OPTIONS` — `strip_tracking_params`
(bool, default `false`), `trailing_slash` (`keep` default), `extra_tracking_params` (list). Адаптеры оборачивают
`UrlNormalizer` при включённой опции (фабрика `UrlNormalizerFactory::fromConfig`). По умолчанию выключено: нормализатор
не должен угадывать канонический вид; документация объясняет, когда включать.

### 4.6. Модель правил: `Condition` против `ParamValue`

- Новый интерфейс `Attribute\Param\Condition` для `when`; `Equals` реализует только его; `ParamValue` остаётся
  «закрытым множеством» из четырёх (docblock синхронизирован). `IndexNow::__construct(when: string|Condition|null)`;
  `Equals` в `params` становится ошибкой типов (phpstan/IDE) и рантайм-ошибкой `ParamExtractor` с текстом из §4.7.
  Breaking для тех, кто передавал `Equals` в `params` (это и был баг). Тир Implement: пользовательские `Condition`
  разрешены (`evaluate(object $subject): bool`).
- `ExplainRunner` печатает фактические значения: `when: status ("draft") -> true — a non-empty string is truthy; use
  new Equals('status', 'published')`; для `params` — значение до и после форматирования.
- `attribute-reference.md` «Anti-patterns» (5 сниппетов); `.phpstorm.meta.php` (§2.3).

### 4.7. Тексты ошибок (12 переписываются)

`Config.php:192` (resolver — два отдельных сообщения с фактами), `:214` (retry — пять отдельных проверок), `:245`
(engines — перечислить допустимые), `:198` (key_prefix — расшифровать ограничение); `ClassNameResolver:46,49`
(где искали; для чего нужен класс); `ParamExtractor:88` (что допустимо, `Equals` только в `when`), `:129` (перечислить
перебранные имена); `ModelLoader:80`, `ActiveRecordLoader:70` (какой базовый класс нужен); `IndexNowObserver
(yii2):229` (что делать); `Checker:128` (начало тела). Правило для всех: факт + допустимое + как исправить.

### 4.8. Паритет наблюдаемости адаптеров

- Laravel и Yii2 передают PSR-14/событийный диспетчер в `Submitter` (Laravel: `Event::dispatch` мост → Telescope видит
  `Result`; Yii2: `trigger()` события компонента `IndexNowComponent::EVENT_RESULT`).
- Laravel `AboutCommand::add('IndexNow', …)`: enabled/dry_run, dispatch, key file URL (маска), debounce store.
- `indexnow:status` (console, новый раннер + команды в трёх адаптерах): коллектор (сколько собрано), debounce store
  (тип, живость), счётчик 403 (§4.3), последняя успешная отправка (через `SubmissionStoreInterface`, если не Null),
  очередь (адаптер: длина, если знает), `--json`.

### 4.9. Прочее волны E

- `explain --json`.
- `Yii2`: `-v/-vv/-vvv` Yii наследуются в `ConsoleOutput` вместо собственного `--verbose`.
- `Config`: явное `dry_run: false` отличается от «не задано» (для будущего расширения авто-dry_run в 1.0; в 0.7 только
  хранится как `?bool`); `production_environments` промах — строка `environment` в check (§4.1) закрывает.
- `docs/bc.md` в каждом адаптере (что стабильно: конфиг-ключи, команды, service id/binding'и/свойства).

### 4.10. Релизы волны E

`core@0.7.0` → `console@0.2.0` → `testing@0.1.1` (если затронут) → `sitemap@0.2.1` → `doctrine@0.4.1` →
`symfony-bundle@0.8.0` → `laravel@0.9.0` → `yii2@0.7.0`. Раздел Changed каждого адаптера: `check` может стать красным
на стейджинге (это цель), `Equals` в `params` больше не принимается.

## 5. Волна F: новые опциональные пакеты

### 5.1. `indexnowkit/history` 0.1.0

- Реализации `SubmissionStoreInterface`: `Psr16SubmissionStore` (кольцевой буфер N записей в PSR-16, дефолт 500),
  `PdoSubmissionStore` (таблица `indexnow_submissions`: url, host, status, reason, http_code, engine, at; индекс по url и
  at; ретенция `purge(olderThan)`); миграции — примеры для Doctrine Migrations, Laravel, Yii2 в `docs/`.
- `History\Console\HistoryRunner` + `Definitions::history()`: `indexnow:history [--host] [--status] [--url] [--limit] [--json]`.
- Строка в `check` через `OptionalPackage` (не установлен) или «history: 1 240 records, last 3 min ago».
- Symfony: вкладка в профилере (последние N); Laravel/Yii2 — команда.
- Адаптеры: `suggest`, регистрация за `OptionalPackage` (образец — sitemap).

### 5.2. `indexnowkit/verify` 0.1.0

- `Verify\VerifyingSubmitter implements SubmitterInterface` — декоратор с pre-flight GET по каждому URL (не HEAD):
  статус, `<meta name="robots">`, `X-Robots-Tag`, `<link rel="canonical">`, robots.txt (кэш на хост). Политики:
  `redirect: skip|follow` (follow — отправить `Location`), `non_canonical: skip|replace`, `origin_error: skip|send`.
  Новые `Reason` добавляются в core в волне E (`Noindex`, `RobotsDisallowed`, `NonCanonical`, `Redirected`,
  `OriginError` — `bc.md` разрешает расширять `Reason`), чтобы пакет не требовал core-релиза.
- Выключен по умолчанию; конфиг-блок `verify` (`enabled`, `timeout`, `concurrency`, `delay` — задержка после коммита,
  чтобы не ловить ложные 404 до деплоя кэша). Документация про стоимость (один GET на URL) и про то, что для
  sitemap-прогонов его надо выключать (`--no-verify` у `sitemap`).
- `check --sample` (§4.1) переиспользует тот же анализатор HTML/заголовков: он живёт в core как `Url\PageSignals`
  (парсер meta robots / canonical / X-Robots-Tag, ~120 строк, без зависимостей), `verify` — только политика и декоратор.

## 6. Политики и критерии 1.0

- **PHP:** минимальная версия поднимается в первом миноре после выхода предыдущей из security-поддержки: `^8.2` → `^8.3`
  в первом миноре после 2026-12-31. Записать в `core/docs/bc.md` и `91-roadmap.md`; таблица совместимости
  `php/docs/compatibility.md` (пакет × PHP × фреймворк × EOL).
- **Фреймворки:** Symfony 6.4 LTS до конца его поддержки; Laravel — два последних мажора; Yii2 2.0.45+; Doctrine ORM
  2.19/3, DBAL 3/4 — до EOL ORM 2.
- **Критерии 1.0 core** (в `91-roadmap.md`): ноль `Symfony\`/`PHPUnit\` в `src/`; ноль лживых `suggest`; тир «may grow»
  исчез (семь интерфейсов зафиксированы); `ParamExtractor::registerReader()` заменён инъекцией; `RuleAwareUrlResolverInterface`
  закрыт; `Adapter\Services` имеет второго потребителя (Yii3) или возвращён в yii2; два минора подряд без breaking;
  `SubmissionStoreInterface`, `Condition`, `FailureCounterInterface` прожили минимум один минор; docs-сайт и AGENTS.md
  есть; conformance-киты в `testing` покрывают C01–C22, A01–A21, H01–H06.
- **BC адаптеров:** `docs/bc.md` в каждом (волна E).
- **Trademark «IndexNow».** Открытый пункт спеки 91 висит после публикации. Решение пользователя; предлагаемая
  формулировка для `00-overview.md` и README: описательное (nominative) использование названия протокола в имени
  `indexnowkit`, дисклеймер уже стоит; при претензии правообладателя — переименование vendor'а с `replace` в composer.
  Зафиксировать решение, а не вопрос.
- **Домен `indexnowkit.dev`:** не блокер; GitHub Pages сейчас, домен по желанию.
- **Спонсорство:** нет; убрать из спеки 90.
- **Flex-рецепт:** PR в recipes-contrib отложен пользователем; в README бандла раздел «Advanced install» с private
  endpoint `extra.symfony.endpoint` на `recipe/index.json` сплита (Flex поддерживает произвольные endpoint'ы) — даёт
  автоустановку сегодня.

## 7. Риски

- **Два новых пакета за одну волну (D).** Механика та же, что у sitemap; риск — в порядке релиза (adapters требуют
  `console ^0.1`, которого нет на Packagist до тега) — `packagist-wait` между шагами уже есть; `split.yml` стадии
  расширяются на testing/console.
- **Переименование assertion-хелперов и `ResultSummary`/`SubmitterFactory`.** Потребители — только монорепо; для
  внешних авторов адаптеров — строка миграции в CHANGELOG.
- **`check` становится красным на стейджинге.** Это цель, но ломает чьи-то пайплайны, где `check` стоял без `--strict` и
  окружение «staging» слало нарочно. Выход: `dry_run: false` явно + `INDEXNOW_ALLOW_NON_PRODUCTION_SUBMISSIONS=1`?
  Нет — не вводить обходной флаг; error только при `enabled && !dryRun && !isProduction`, и в CHANGELOG сказано, как
  выключить (`dry_run`, `enabled`, или добавить окружение в `production_environments`, если оно действительно боевое).
- **`Condition`/`ParamValue` — breaking в модели атрибутов.** Затронуты только те, кто передавал `Equals` в `params`
  (баг); `AttributeTest` и conformance подтверждают остальное.
- **`--sample` ходит по сайту пользователя.** Только по явному флагу; таймаут 5 с; не в `--json` без флага.
- **Docs-сайт и монорепо-сборка.** Ссылки в README относительные к пакету (`docs/...`) и абсолютные на GitHub;
  сборщик переписывает их плагином, тест сайта проверяет отсутствие битых ссылок (`mkdocs build --strict`).
- **AGENTS.md разъезжается с кодом.** `AgentsFileTest` проверяет команды/ключи; таблица ключей генерируется.
- **Объём.** Волна 0 не требует релизов core и делается параллельно с D; E — самая большая, но каждая строка §4 —
  отдельный коммит с тестом.

## 8. Definition of Done

Волна 0:
- Packagist-витрина трёх пакетов актуальна; `bin/packagist-check` в репо; topics/LICENSE/Issues-off/org `.github`/
  Dependabot/CodeQL/CoC/шаблоны на месте; бейджи и `authors` едины.
- README-дефекты §2.2 исправлены; `ReadmeQuickstartTest` в трёх адаптерах зелёный.
- `AGENTS.md` ×7, `context7.json`, Boost guidelines, `.phpstorm.meta.php`; `AgentsFileTest` зелёный.
- Docs-сайт опубликован, `llms.txt`/`llms-full.txt` доступны, `mkdocs build --strict` в CI.
- Документы §2.5–2.7 написаны; Yii2-доки по объёму разделов не уступают Laravel; три RU-перевода.

Волна D:
- `grep -rl 'Symfony\\\|PHPUnit\\' packages/core/src` пуст; `composer.json` core без `symfony/console`, `phpunit` в
  `suggest`/`require-dev`; пакеты `testing` и `console` на Packagist; все адаптеры зелёные на `highest`/`lowest`.
- `grep -rn 'SitemapSupport\|class_exists(\\\\IndexNowKit\\\\Sitemap' packages/{symfony-bundle,laravel,yii2}/src` пуст
  (всё через `OptionalPackage`).
- Тест дефекта дебаунса; `Engine` с двумя новыми участниками; `INDEXNOW_PREVIOUS_KEY`.

Волна E:
- `indexnow:check --json --strict` во всех трёх адаптерах; тест «staging + key → error»; `Content-Type`, `Cache-Control`,
  robots, `previous_key`, `--sample` покрыты тестами с `FakeTransport`; `DebounceStoreCheck` в бандле.
- `key:generate --force` пишет `PREVIOUS_KEY`; `FailureCounterInterface` с PSR-16 реализацией; `SubmissionStoreInterface`
  + `NullSubmissionStore` подключены во всех адаптерах; `CanonicalUrlNormalizer` + опции; `Condition`; 12 текстов;
  `explain` со значениями и `--json`; `status`; `about`; PSR-14 в Laravel/Yii2; `docs/bc.md` в адаптерах.

Волна F:
- `history` и `verify` на Packagist, зарегистрированы в адаптерах за `OptionalPackage`, документированы, строки в `check`.

Повторный аудит по пяти линзам после F: все ≥ 9, кроме «DX AI» ≥ 8 (зависит от внешних индексаторов).

## 9. Решения, которые остаются за пользователем

1. Лицензия `indexnowkit/spec`: CC-BY-4.0 (дефолт) или MIT.
2. Trademark: принять формулировку §6 и закрыть пункт 91.
3. Домен `indexnowkit.dev`: покупать сейчас или позже (дефолт — позже).
4. RU-перевод: три документа (дефолт) или вся документация.
5. Волна F (`history`, `verify`): делать сразу после E (дефолт) или по спросу.
6. Спонсорство: не заводить (дефолт).
