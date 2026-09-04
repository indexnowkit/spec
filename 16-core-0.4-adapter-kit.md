# 16. core 0.4 / 0.5 «adapter kit»: вынос sitemap, конфиг-фабрика, общие блоки адаптеров

Статус: спецификация v2 (после адверсального ревью 2026-09-04); волны A (core 0.4.0) и B (core 0.5.0) выпущены 2026-09-05, волна C (core 0.5.1, sitemap снова `suggest`) выпущена 2026-09-05 — отклонения в разделах «Уточнения по реализации». Основание: аудит core 0.3.1 и четырёх
адаптеров (Yii2 как контрольный «новый адаптер на готовом core»). Цель: авторинг адаптера с 6 до 9, API/тесты/документация
с 8 до 9, без потери 9 по протоколу, надёжности и безопасности.

Волна разбита на два минора core, каждый с релизом всех адаптеров:

| Волна | core | Содержание | Адаптеры |
|---|---|---|---|
| A | 0.4.0 | §1 sitemap-пакет, §2 `Config` + `Adapter\ConfigFactory`, §3.1 статические фабрики, §4.1 мелкие блоки, phpstan lowest | doctrine 0.3.0, symfony-bundle 0.4.0, laravel 0.5.0, yii2 0.2.0, sitemap 0.1.0 |
| B | 0.5.0 | §3.2 `Adapter\Services`, §4.2 `Hook\ObserverHelper`, §4.3 `Retry\WorkerOutcome`, §4.4 `Console\Definitions`, §5 assertion-хелперы и coverage | doctrine 0.3.1, symfony-bundle 0.5.0, laravel 0.6.0, yii2 0.3.0, sitemap 0.1.1 |
| C | 0.5.1 | §1.5/§1.7: `indexnowkit/sitemap` в адаптерах снова `suggest` (решение пользователя 2026-09-05), `Check\StaticCheck`, `ConfigFactory(ignoreBlocks:)` | symfony-bundle 0.6.0, laravel 0.7.0, yii2 0.4.0 |

Волна A — механическая и низкорисковая (перенос, фабрики над существующими конструкторами). Волна B меняет форму
адаптеров и требует проектного внимания; она начинается только после релиза A.

## 0. Диагноз

- `IndexNowComponent` 600 строк, `IndexNowKitServiceProvider` 380, `IndexNowKitLoader` 380; около 40% — один и тот же код в
  трёх экземплярах: `ConfigFactory` ×3 (вырезать свои блоки, алиас `serve_key_file`, `environment`, проверка `dispatch`,
  «queue требует base_url», warning на неизвестные ключи, `critical` + disabled при ошибке), сборка графа ×3
  (`LazyTransport` с резолвом `http.client`, `UrlNormalizer`, `TokenBucket`, `Client`, debounce-store, `Submitter`,
  `Collector`, dispatcher, `AttributeUrlResolver` семью позиционными аргументами, `SitemapReader` девятью со своим `intOf`,
  `SubmitterFactory` семью).
- Observer'ы Laravel и Yii2 совпадают в guard/deliver/WeakMap отложенных удалений/текстах; различается только граница коммита.
- `SubmitUrlsJob` laravel/yii2 и `SubmitUrlsHandler` messenger трижды считают «что ретраить, что финально, какой текст» —
  тремя разными способами (Laravel: `release(delay)`, Messenger: `RecoverableMessageHandlingException` с максимальным
  Retry-After, Yii2: `canRetry()` без задержки).
- `SubjectLoader` ×3 повторяют `resolveClass`; `CacheCheck`/`QueueCheck` laravel/yii2 — одна логика, разные фразы;
  `ContainerResolverLocator` ×3; `urlsForAll`/`submitMany` в двух менеджерах.
- Sitemap (≈1100 строк, 12% core, единственная причина `ext-xmlreader`/`ext-zlib`, единственное место, где core ходит по
  URL из чужого документа) протёк в фасад (`sitemap()`, `$sitemap`), `Console\Vocabulary`, `Console\ResultRenderer`,
  `Check`, docblock'и транспорта, `composer.json` (description, keywords, suggest), README, SECURITY, bc.md, adapters.md.
- Ошибка во всех трёх адаптерах: в allowed-список `unknownOptions()` попадает голый ключ `key_file` (и `sitemap`), и
  `Config::unknownOptions()` (`Config.php:595-599`) перестаёт проверять вложенные ключи блока — опечатка `key_file.enabld`
  проходит молча.
- API-шероховатости: `CheckReport::ok/warning/error` помечены `@internal`, хотя `CheckInterface` обязывает их звать;
  `GuardedUrlResolver` трижды проверяет `instanceof AttributeUrlResolver` — свой rule-aware резолвер невозможен; сдвоенные
  docblock у `Client::FORBIDDEN_ESCALATION`, `Collector::__construct`, `CallableDispatcher::__construct`;
  `IndexNowKit::collect()` считает размер через копию массива; `Result::urlsOf()` deprecated с 0.2.0.
- Тесты: H01–H06 каждый адаптер пишет с нуля; coverage не измеряется; phpstan только на highest.
- Документация: core README и Install знают только Symfony/Doctrine; adapters.md §17 «core ^0.2», §18 без yii2; bc.md
  «pin ^0.2.0»; спека 02 упоминает `MapKeyProvider` (нет в коде) и throttle «только в queue» (в коде всегда в `Client`),
  в схеме конфига нет `key_file`; спека 10 «PHP 8.2–8.4»; у yii2 нет `docs/troubleshooting.md` и раздела Debugging
  (у laravel есть).
- Релизный инструментарий: `tag.sh`, `packagist-wait.sh`, `release-notes.py`, `newkey.sh` живут в scratchpad сессии, не в
  репо; в репо есть только `tools-push-splits.sh` и `.github/workflows/split.yml`.

Что НЕ меняем: протокольный слой (`Client`, `Result`, `Reason`, `Submitter`), модель правил (`UrlRule`, `RuleCompiler`,
`ChangeClassifier`, `ObjectChangeHandler`), staging (`TransactionStaging`, `VerifyingStaging`), транспорт, граница коммита в
адаптерах. `ParamExtractor::registerReader()` (статическая регистрация) — долг до 1.0, в этой волне не трогается.

## 1. Пакет `indexnowkit/sitemap` (волна A)

Принцип: **sitemap — потребитель core, а не его часть.** В core не остаётся ни кода, ни опций, ни типов, ни текстов
про sitemap. Core знает о нём ровно как о любом другом пакете семейства: строкой в таблице «Family» README и записью в
CHANGELOG.

### 1.1. Что переезжает

| Было (core 0.3) | Стало (sitemap 0.1) |
|---|---|
| `IndexNowKit\Sitemap\{SitemapReader, Spool, SpoolMode, SitemapEntry, SitemapSourceInterface}` | те же FQCN, пакет `sitemap` (Composer PSR-4 longest-prefix: `IndexNowKit\Sitemap\` в другом пакете безопасен после удаления `core/src/Sitemap`) |
| `IndexNowKit\Console\SitemapRunner`, `SitemapOptions` | `IndexNowKit\Sitemap\Console\SitemapRunner`, `SitemapOptions` |
| `IndexNowKit\Check\SitemapSpoolCheck` | `IndexNowKit\Sitemap\Check\SitemapSpoolCheck` |
| `tests/Unit/{SitemapReaderTest, SitemapReaderIntegrationTest, SpoolTest, SitemapSpoolCheckTest}`, sitemap-часть `Console/RunnersTest`, sitemap-часть `FacadeTest` | `packages/sitemap/tests`; копируются нужные хелперы харнесса (`Factory::config()`, `io()`, `submitters()`; `FakeTransport` уже публичный в `Testing\`) |
| sitemap-сценарии `tests/Support/mock-server/router.php` | `packages/sitemap/tests/Support/mock-server/router.php` — свой роутер, свой bootstrap поднимает встроенный сервер |
| README §«Sitemap\SitemapReader streams…», `retries-and-queues.md` «Prefer the sitemap path», SECURITY (allowForeignHosts, 50 MiB), `bc.md` строки про `SitemapReader`/`Spool`/`SitemapEntry`/`MAX_*`, `adapters.md` строка команды `sitemap` и §«Sitemap» | README, SECURITY.md, `docs/bc.md`, `docs/adapters.md` пакета sitemap |
| `composer.json`: «sitemap reader» в description, keyword `sitemap`, suggest `ext-xmlreader`/`ext-zlib`, упоминание в suggest `symfony/console` | `require ext-xmlreader`, `suggest ext-zlib` в пакете sitemap |

### 1.2. Что удаляется или переформулируется в core

- `IndexNowKit::sitemap()`, поле `$sitemap`, параметр `$sitemap` конструктора и `create()`, `use IndexNowKit\Sitemap\*`.
- `Console\Vocabulary::$sitemapUrlOption` (уходит аргументом в `Sitemap\Console\SitemapRunner`).
- `Console\ResultRenderer::summary()`: `'Nothing submitted: the sitemap yielded no URL.'` → `'Nothing submitted: the source
  yielded no URL.'`; docblock'и `ResultSummary`/`ResultFormatterInterface::summary()`: «aggregated results of a run that
  submits in batches (a large subject set, a streamed source)». `ResultSummary` остаётся в core: он общий для любых
  многобатчевых прогонов (`submitAll()` §4.1 тоже его использует).
- `Http\TransportInterface::get()` docblock «GET a document in full (key file, sitemap)» → «(a key file, any document a
  consumer reads)»; `StreamingTransportInterface` — «consumers that stream large documents use it when available»;
  `Psr18Transport` — «50 MiB, a generous cap for the largest documents consumers of this transport read» и
  `$getBodyLimit bytes of a GET body (key files, streamed documents)`.
- Тестовый роутер core: маршруты `/sitemap.xml` и `/sitemap.xml.gz` переименовываются в `/large-document.xml[.gz]`; их
  единственный потребитель в core — тесты GET-лимита `Psr18TransportUnitTest`, URL там тоже переименовываются.
  `AttributeUrlResolverTest` заменяет литерал `/sitemap.xml` на `/feed.xml`.
- `composer.json`: description «… key file handling. Framework-agnostic; adapters for Doctrine, Symfony, Laravel, Yii2, a
  sitemap add-on» — единственное разрешённое упоминание вне README-таблицы; keywords без `sitemap`; suggest без
  `ext-xmlreader`/`ext-zlib`; suggest `symfony/console` — «check/submit/explain/key commands».
- README (EN/RU): intro без «sitemap reader», раздел Sitemap → строка в таблице «Family»: `indexnowkit/sitemap` —
  «reads a sitemap (index, gzip, text) and submits its URLs; the `sitemap` command of every adapter». Google-абзац
  «keep your sitemap for Google» остаётся: это про протокол, не про пакет.
- CHANGELOG core 0.4.0: раздел «Removed» с миграцией (§9).

Гейт (DoD): `grep -ril sitemap packages/core --exclude-dir=vendor --exclude-dir=.phpunit.cache
--exclude=composer.monorepo.json` возвращает ровно `CHANGELOG.md`, `README.md`, `README.ru.md`, `composer.json`,
`docs/bc.md`, `docs/adapters.md`; в `composer.json` — одно вхождение (description), в README EN/RU — строка таблицы
Family плюс Google-абзац (два слова «sitemap» про протокол, не про пакет); в `docs/bc.md` — миграционная заметка
(что переехало в 0.4.0); в `docs/adapters.md` — ссылки на пакет как на пакет семейства (require, `SitemapConfig::OPTIONS`
в `ownedOptions`, `SitemapSpoolCheck` в чек-листе DoD). Остальные документы core (`docs/operations.md`,
`docs/configuration.md`, SECURITY и другие) говорят «пакет-дополнение из таблицы семейства», не называя его; лог-строка
`invalid sitemap configuration…` документируется в `docs/adapters.md` пакета sitemap, не в core.

Адаптеры при `sitemap.enabled: false`: бандл не регистрирует команду; Laravel и Yii2 печатают `sitemap.enabled is
false.` и выходят с `INVALID` (Laravel раньше игнорировал флаг в команде). Невалидный блок `sitemap` в Laravel/Yii2 —
critical-лог и `SitemapConfig::disabled()`, как у core-опций.

### 1.3. Новое в пакете

```php
namespace IndexNowKit\Sitemap;

final readonly class SitemapConfig
{
    /** Только dotted-ключи, никогда голый `sitemap` (иначе Config::unknownOptions() перестаёт видеть вложенные опечатки). */
    public const OPTIONS = ['sitemap.enabled', 'sitemap.url', 'sitemap.max_depth', 'sitemap.max_sitemaps',
        'sitemap.max_bytes', 'sitemap.allow_foreign_hosts', 'sitemap.spool', 'sitemap.spool_dir', 'sitemap.fetch_retries'];

    public function __construct(
        public bool $enabled = true,
        public ?string $url = null,                 // абсолютный http(s), локальный путь или file://
        public int $maxDepth = 3,
        public int $maxSitemaps = SitemapReader::MAX_SITEMAPS,
        public int $maxBytes = SitemapReader::MAX_XML_BYTES,
        public bool $allowForeignHosts = false,
        public SpoolMode $spool = SpoolMode::Auto,
        public ?string $spoolDir = null,
        public int $fetchRetries = 2,
    ) {}                                            // валидация как у Config: диапазоны, ConfigurationException с именем ключа

    /** @param array<string, mixed> $block сырой блок `sitemap` адаптера; числа/булевы как в Config::fromArray() (is_numeric, "1"/"0"/"true"/"false") */
    public static function fromArray(array $block): self;
    public static function disabled(): self;
}

final class SitemapReader implements SitemapSourceInterface
{
    public static function fromConfig(SitemapConfig $config, TransportInterface $transport, LoggerInterface $logger = new NullLogger()): self;
    // конструктор без изменений
}

namespace IndexNowKit\Sitemap\Console;

final class SitemapRunner
{
    public function __construct(
        IndexNowKit $indexNow,
        SitemapSourceInterface $reader,
        SubmitterFactoryInterface $submitters,
        ?string $defaultSitemap = null,
        ResultFormatterInterface $formatter = new ResultRenderer(),
        string $sitemapUrlOption = 'sitemap.url',   // бывшее Vocabulary::$sitemapUrlOption; печатается в «Give a sitemap URL, or configure %s or base_url.»
    ) {}                                            // Vocabulary не принимает: раннер не печатает ни одного слова из него (phpstan: unused property)
}

namespace IndexNowKit\Sitemap\Check;

final class SitemapSpoolCheck implements CheckInterface
{
    public function __construct(SitemapConfig $config) {}   // вместо сырого массива (три call-site'а в адаптерах)
    // инвариант: enabled === false → check() не пишет ни одной строки
}
```

Транспорт для читателя берётся у адаптера (тот же, что у `Client`): `$kit->transport` публичный, но `null` при
кастомном `$submitter` в `create()` — тогда `Http\TransportFactory::lazy($kit->config)` (§3.1).

### 1.4. `composer.json` пакета

`name: indexnowkit/sitemap`, `type: library`, `description: "Sitemap reader for indexnowkit: streams sitemap indexes,
gzip and text sitemaps into IndexNow submissions; the sitemap command of every adapter"`, `require: php ^8.2,
ext-xmlreader, indexnowkit/core ^0.4` (`^0.4` уже исключает core 0.3 с дублирующими FQCN — отдельный `conflict` не
нужен), `suggest: ext-zlib (gzip-compressed sitemaps), symfony/console (the sitemap command body)`, `require-dev:
symfony/console, phpunit, phpstan + strict-rules, psr/log`, `autoload IndexNowKit\Sitemap\ => src/`, `branch-alias
dev-main 0.1.x-dev`, `minimum-stability dev + prefer-stable`, scripts `ci:install:highest|lowest` как у адаптеров.
`ext-xmlreader` в `require`: без него `SitemapReader` бесполезен, сейчас он падает `TransportException` в рантайме.

### 1.5. Адаптеры

Решение v2 «`require`, не `suggest`» отменено пользователем 2026-09-05: sitemap ставится только по необходимости.
Волны A и B вышли с `require`; возврат к `suggest` — волна C (§1.7). Ниже — целевое состояние.

- `suggest: indexnowkit/sitemap` (`"the indexnow:sitemap command"`), не `require`. Без пакета всё остальное работает
  без единого warning'а в логах; диагностика — в `check`.
- Определение «пакет установлен»: один предикат на адаптер (`class_exists(\IndexNowKit\Sitemap\SitemapReader::class)`),
  переопределяемый для тестов (бандл: параметр конструктора `IndexNowKitConfiguration`/`IndexNowKitLoader`
  `bool $sitemapInstalled`; Laravel/Yii2: `@internal` статическое поле `SitemapSupport::$installed`).
- Вся sitemap-проводка адаптера живёт за предикатом и читает только `SitemapConfig::fromArray($block)`: reader через
  `SitemapReader::fromConfig`, `SitemapSpoolCheck(SitemapConfig)`, `SitemapRunner(..., sitemapUrlOption: '<префикс>.sitemap.url')`.
  Ни одного `use IndexNowKit\Sitemap\*` в файлах, которые загружаются без пакета (иначе `SitemapConfig::OPTIONS`,
  `SitemapReader::MAX_*`, `Sitemap\Console\Definitions` падают fatal'ом): sitemap-код адаптера — в отдельных классах
  (`<Adapter>\Sitemap\SitemapServices`, `<Adapter>\Console\SitemapCommand`), которые инстанцируются только при
  `installed`.
- `unknownOptions`: allowed = `Config::OPTIONS` + свои dotted-ключи + (`SitemapConfig::OPTIONS`, если установлен).
  Без пакета блок `sitemap` в конфиге не даёт warning «unknown option»: `Adapter\ConfigFactory` получает параметр
  `ignoreBlocks: list<string>` (адаптер передаёт `['sitemap']`, когда пакета нет) — ключи этих блоков пропускаются целиком.
  Голые `key_file`/`sitemap` в `ownedOptions` по-прежнему запрещены (§0).
- Команда `sitemap` без пакета существует и объясняет: `indexnow:sitemap` (бандл, Laravel) — заглушка
  `SitemapNotInstalledCommand` с тем же именем, `ignoreValidationErrors()`, вывод
  `indexnowkit/sitemap is not installed: composer require indexnowkit/sitemap` и exit `ExitCode::FAILURE`; Yii2
  `actionSitemap` печатает то же. Так cron после обновления получает понятную ошибку, не «command not found».
- `check` без пакета печатает одну строку `Check\StaticCheck` (core, новый): `sitemap: not installed (composer require
  indexnowkit/sitemap)`; если блок `sitemap` в конфиге непустой — `sitemap: not installed, the sitemap block in the
  configuration is ignored (composer require indexnowkit/sitemap)`. Уровень ok: отсутствие опционального пакета — не проблема.
- Бандл: узел `sitemap` дерева строится через `SitemapReader::MAX_*`/`SpoolMode` только при `installed`; иначе
  `->arrayNode('sitemap')->ignoreExtraKeys(false)` (узел есть, старые yaml компилируются, содержимое игнорируется).
  Сервисы `indexnowkit.sitemap_config`, `sitemap_reader`, `check.sitemap_spool`, `console.sitemap`, `SitemapCommand` —
  только при `installed && sitemap.enabled`; иначе `check.sitemap_missing` (`StaticCheck`) и `SitemapNotInstalledCommand`.
- Laravel: binding `SitemapSourceInterface` и остальные sitemap-singleton'ы — только при `installed`; иначе заглушка команды
  и `StaticCheck`. Опубликованный `config/indexnow.php` сохраняет блок `sitemap` с комментарием «needs indexnowkit/sitemap».
- Yii2: `sitemapConfig()`/`sitemapSource()` остаются публичными; без пакета бросают `LogicException` с текстом установки;
  `IndexNowController::definitions()` включает `sitemap` только при `installed`; `actionSitemap` без пакета — ошибка и
  `ExitCode::FAILURE`.
- Тесты в каждом адаптере с предикатом `false`: команда-заглушка, строка `check`, блок `sitemap` в конфиге без warning'а,
  остальные команды работают; с предикатом `true` — текущие тесты.
- README адаптеров: Install — `composer require indexnowkit/sitemap   # optional: the indexnow:sitemap command`;
  раздел Sitemaps начинается с этой строки.

### 1.6. Спека и документация

02: строка «Отправить sitemap» в таблице публичного API → «пакет `indexnowkit/sitemap`»; в схеме конфига блок `sitemap`
с пометкой «пакет sitemap», блок `key_file`. 10: пространство `Sitemap\` в отдельный подраздел «пакет sitemap». 90: пакет в
семействе и в таблице README-шаблона. 91: статус. Порты на другие языки повторяют форму: core + sitemap-аддон.

### 1.7. Волна C: sitemap снова опциональный (core 0.5.1, адаптеры)

| Пакет | Версия | Изменения |
|---|---|---|
| `core` | 0.5.1 | `Check\StaticCheck(CheckLevel $level, string $line)` (Call); `Adapter\ConfigFactory(..., array $ignoreBlocks = [])`; `docs/adapters.md` §2/§20: «опциональные пакеты: предикат, заглушка команды, StaticCheck, ignoreBlocks» |
| `sitemap` | — | без изменений |
| `doctrine` | — | без изменений |
| `symfony-bundle` | 0.6.0 | `require` → `suggest`; `IndexNowKitConfiguration`/`IndexNowKitLoader` с `$sitemapInstalled`; `SitemapNotInstalledCommand`; `StaticCheck`; sitemap-сервисы за предикатом; тесты с предикатом false |
| `laravel` | 0.7.0 | то же: `SitemapSupport`, `Sitemap\SitemapServices` (регистрация singleton'ов), заглушка, `StaticCheck`, `ConfigFactory` с `ignoreBlocks` |
| `yii2` | 0.4.0 | то же: `SitemapSupport`, guard в `sitemapConfig()/sitemapSource()`, `definitions()` условно, `actionSitemap` без пакета |

Миграция (в CHANGELOG каждого адаптера, раздел Changed, первым пунктом): «`indexnowkit/sitemap` больше не ставится
автоматически. Если вы используете `indexnow:sitemap` — `composer require indexnowkit/sitemap`, иначе после
`composer update` команда сообщит об отсутствии пакета и завершится с кодом 1». Порядок релиза: `core@0.5.1` →
`packagist-wait` → `symfony-bundle@0.6.0` → `laravel@0.7.0` → `yii2@0.4.0`.

DoD волны C: `bin/ci` зелёный; в каждом адаптере тест-набор с предикатом `false` (команда-заглушка, строка `check`, блок
`sitemap` без warning'а, остальные команды работают); `grep -rn 'IndexNowKit\\Sitemap' packages/{symfony-bundle,laravel,yii2}/src`
даёт только файлы, инстанцируемые за предикатом (список в спеке тем же коммитом); `composer.json` адаптеров без
`indexnowkit/sitemap` в `require`; README/CHANGELOG обновлены; спека 12/13/15 и 91 синхронизированы.

#### Уточнения по реализации (волна C, реализована 2026-09-05)

Файлы, в которых встречается `IndexNowKit\Sitemap` (гейт §1.7), и почему каждый безопасен без пакета:

| Адаптер | Файл | Почему безопасен |
|---|---|---|
| symfony-bundle | `DependencyInjection/SitemapServices.php` | предикат `installed()` (`class_exists` на `::class`-константе — автозагрузки нет), `configure()`/`register()` вызываются только при `installed` |
| symfony-bundle | `Command/SitemapCommand.php` | сервис регистрируется только из `SitemapServices::register()` |
| laravel | `Sitemap/SitemapSupport.php` | только `class_exists(\IndexNowKit\Sitemap\SitemapReader::class)` |
| laravel | `Sitemap/SitemapServices.php` | `use` не грузит классы; `options()`/`register()` вызываются только при `installed` |
| laravel | `Console/SitemapCommand.php` | регистрируется в artisan только через `SitemapServices::commands()` |
| yii2 | `Sitemap/SitemapSupport.php` | только `class_exists(...)` |
| yii2 | `Sitemap/SitemapServices.php` | статические методы, вызываются только при `installed` |
| yii2 | `Console/SitemapAction.php` | `definition()`/`run()` — только при `installed` |
| yii2 | `IndexNowComponent.php` | `use` для типов возврата `sitemapConfig(): SitemapConfig` / `sitemapSource(): SitemapSourceInterface`; оба метода бросают `LogicException` до обращения к типам, `checks()` создаёт `SitemapSpoolCheck` через `SitemapServices` только при `installed` |

Отклонения от текста §1.5/§1.7:

- Дефолт предиката в бандле: `?bool $sitemapInstalled = null` у `IndexNowKitBundle`, `IndexNowKitConfiguration` и
  `IndexNowKitLoader` (null = `SitemapServices::installed()`); `bool = class_exists(...)` в сигнатуре PHP невозможен.
  `IndexNowKitConfiguration::build()` стал методом экземпляра. Тесты: `new IndexNowKitBundle(sitemapInstalled: false)`
  в `TestKernel` (варианты `nositemappkg`, `nositemappkgcfg`).
- Бандл при `installed && !sitemap.enabled` сохраняет прежнее поведение (нет команды, есть `sitemap_config` и
  `check.sitemap_spool`, который для disabled ничего не печатает), а не заглушку «not installed»: пакет-то установлен.
  Заглушка и `check.sitemap_missing` — только при `!installed`.
- Laravel: блок `sitemap` всегда присутствует в конфиге (пакетный `config/indexnow.php` через `mergeConfigFrom`),
  поэтому вариант «block is ignored» печатается, когда блок отличается от пакетных дефолтов
  (`SitemapSupport::checkLine($block, $defaults)`); совпадающий с дефолтами блок даёт простую строку.
- Yii2: без пакета `IndexNowController::options('sitemap')` по-прежнему принимает опции команды (список
  `SITEMAP_OPTIONS_WITHOUT_PACKAGE` зеркалит `Sitemap\Console\Definitions::sitemap()`), иначе cron с `--dry-run`
  получал бы «Unknown option» вместо текста установки. Добавлен `IndexNowComponent::sitemapInstalled()`.
- Заглушки печатают текст одной строкой `<error>…</error>` (не блоком `SymfonyStyle::error()`, который переносит
  строки): лог cron'а ищет фразу grep'ом.
- Предикат в Laravel/Yii2 — `SitemapSupport::$installed` типа `?bool` (null = определить, false/true = принудительно).
- `ConfigFactory(ignoreBlocks:)` отвергает dotted-ключи `LogicException`'ом; голые имена блоков в `ownedOptions`
  остаются задокументированным запретом без проверки в конструкторе (бандл передаёт все узлы дерева, включая
  имена блоков, где `unknownOptions()` — no-op).

## 2. `Config` 0.4 и `Adapter\ConfigFactory` (волна A)

### 2.1. Универсальные опции переезжают в core

| Ключ | Свойство `Config` | Тип, дефолт | Семантика |
|---|---|---|---|
| `key_file.enabled` | `serveKeyFile` | bool, `true` | псевдоним `serve_key_file`. `fromArray()`: явный `serve_key_file` выигрывает (как сегодня во всех адаптерах), иначе `key_file.enabled`, иначе `true`; `serve_key_file` документируется как deprecated |
| `key_file.cache_max_age` | `keyFileMaxAge` | int ≥ 0, `KeyFileResponder::DEFAULT_MAX_AGE` | `Cache-Control: max-age` файла ключа |
| `debounce.store` | `debounceStore` | `?string`, `null` | `null` = адаптер выбирает (Laravel `cache`, бандл `cache.app`, Yii2 `cache`; plain PHP `memory`); `memory` \| `none` \| идентификатор, который адаптер резолвит в свой кэш |
| `http.client` | `httpClient` | `?string`, `null` | id/класс PSR-18 клиента; `null` = discovery |

`Config::OPTIONS` дополняется **только dotted-ключами** `key_file.enabled`, `key_file.cache_max_age`, `debounce.store`,
`http.client`. `Config::with()` получает строки для четырёх новых свойств; `Config::fromEnv()` — переменные
`INDEXNOW_KEY_FILE_ENABLED`, `INDEXNOW_KEY_FILE_CACHE_MAX_AGE`, `INDEXNOW_DEBOUNCE_STORE`, `INDEXNOW_HTTP_CLIENT`
(`INDEXNOW_SERVE_KEY_FILE` остаётся). Тесты: `ConfigTest`/`ConfigWithTest`/`ConfigFromEnvTest`/`ConfigValidationTest`
дополняются на каждую опцию, плюс приоритет `serve_key_file` над `key_file.enabled`.

Новое в `Config`: `keyFileHeaders(): array<string, string>` =
`KeyFileResponder::headers($this->keyFileMaxAge, varyHost: $this->hosts !== [])` — три адаптера считают это сами.

Адаптер-специфичные ключи (`key_file.path|host|route_name|middleware|pattern`, `router.*`, `queue.*`, `messenger.*`,
`eloquent.*`, `active_record.*`, `doctrine.*`, `logging.*`, `profiler.*`, `flush.*`) остаются у адаптеров.

### 2.2. `IndexNowKit\Adapter\ConfigFactory`

```php
namespace IndexNowKit\Adapter;

final class ConfigFactory
{
    /**
     * @param list<string>                     $ownedOptions  dotted-ключи адаптера и фича-пакетов сверх Config::OPTIONS
     * @param list<string>                     $dispatchModes допустимые значения `dispatch` кроме `auto`; [0] — дефолт
     * @param (Closure(): string)|null         $autoDispatch  режим для `dispatch: auto`; null = «auto is not supported here»
     * @param list<string>                     $needBaseUrl   режимы, при которых base_url обязателен (воркер без запроса)
     * @param array<string, scalar|array<string, scalar|null>|null> $defaults
     *        значения адаптера поверх дефолтов core: скаляры верхнего уровня и блоки из скаляров (`['dispatch' => 'queue',
     *        'debounce' => ['store' => 'cache']]`). Списки (`engines`, `hosts`, `production_environments`) в defaults
     *        запрещены — конструктор бросает LogicException: их слияние с raw неоднозначно.
     * @param (Closure(Config): ?string)|null  $validate      пост-проверка адаптера (Yii2: «queue component "x" is not
     *        configured»); строка = текст ConfigurationException
     * @param string                           $checkCommand  как вызвать check в этом фреймворке, для critical-лога
     */
    public function __construct(
        private array $ownedOptions = [],
        private array $dispatchModes = ['sync', 'none'],
        private ?Closure $autoDispatch = null,
        private array $needBaseUrl = ['queue', 'messenger'],
        private array $defaults = [],
        private ?Closure $validate = null,
        private string $checkCommand = 'indexnow:check',
    ) {}

    /** Строгий путь (check, тесты, компиляция контейнера): бросает ConfigurationException. */
    public function build(array $raw, ?string $environment): Config;

    /** Безопасный путь (рантайм): warning на неизвестные ключи, critical + disabled() при ошибке. Никогда не бросает. */
    public function load(array $raw, ?string $environment, LoggerInterface $logger): Config;

    /** @return list<string> dotted-ключи raw, которых нет ни в Config::OPTIONS, ни в ownedOptions */
    public function unknownOptions(array $raw): array;

    public static function disabled(?string $environment): Config;   // new Config(enabled: false, dryRun: true, environment: $environment)
}
```

`build()` по шагам:

1. Слияние `defaults` с `raw`: ключ верхнего уровня из raw заменяет дефолт целиком, кроме известных блоков
   (`http`, `debounce`, `key_file`, `throttle`, `retry`, `batch`, `logging` и блоки, чьи dotted-ключи есть в `ownedOptions`),
   которые сливаются по ключу второго уровня. Никакого `array_replace_recursive` — он превращает списки в словари.
2. `Config::fromArray($merged)` с `environment ??= $environment`.
3. `dispatch === 'auto'` → `($autoDispatch)()` или `ConfigurationException('"dispatch" is "auto", which this adapter does
   not support; use one of: sync, none, …')`. Затем `dispatch ∈ dispatchModes`.
4. `dispatch ∈ needBaseUrl && baseUrl === null` → `'"dispatch" is "queue" but "base_url" is not set: a worker has no
   request to take the host from'`.
5. `$validate($config)` → строка = исключение.

`load()` = `unknownOptions` → warning `indexnow: unknown option(s) in the indexnow configuration: {options}`; `build()` в
`try` → при `ConfigurationException` лог `critical` `indexnow: invalid configuration, IndexNow is disabled until it is
fixed: {error} (run "{check}")` и `disabled($environment)`.

Адаптеры: три `ConfigFactory` сжимаются до объявления `Adapter\ConfigFactory` с параметрами; `IndexNowKitLoader` бандла
резолвит `auto` на этапе компиляции (Messenger есть или нет — известно только контейнеру) и передаёт
`autoDispatch: null`, `needBaseUrl: []` (дерево уже проверило), `ownedOptions` = все узлы дерева (обработанное дерево
содержит все ключи с дефолтами, поэтому список полный, и pass на неизвестные — no-op). `dispatch` должен быть уже
`messenger|sync|none`. Тесты: `ConfigFactoryTest` в core (слияние, auto, needBaseUrl, validate, load-путь с логами,
списки в defaults запрещены).

## 3. Фабрики (волна A) и `Adapter\Services` (волна B)

Два слоя, потому что контейнеры бывают двух видов: DI с описанием сервисов (Symfony, Laravel) и рантайм-сборка (Yii2,
Yii3, Битрикс, plain PHP). Первому нужны фабрики на каждый узел, второму — готовый ленивый набор сервисов.

### 3.1. Статические фабрики (слой 1, волна A)

| Фабрика | Что делает | Заменяет |
|---|---|---|
| `Http\TransportFactory::lazy(Config $config, ?Closure $clientLocator = null): LazyTransport` | `httpClient === null` → `Psr18Transport::discover(timeout)`; иначе `Psr18Transport::discover(self::psr18($clientLocator($id), $id), timeout)`; `httpClient !== null` без локатора → `ConfigurationException` | три копии замыкания |
| `Http\TransportFactory::psr18(mixed $instance, string $id): ClientInterface` | «"http.client" "{id}" resolves to {type}, which is not a PSR-18 client» | три копии |
| `Debounce\DebounceStoreFactory::fromConfig(Config, ?Closure $cacheLocator = null, string $default = 'memory'): DebounceStoreInterface` | `debounceStore ?? $default`; `memory` → `MemoryDebounceStore`, `none` → `NullDebounceStore`, иначе `Psr16DebounceStore($cacheLocator($id), keyPrefix)`; без локатора → `ConfigurationException` «"debounce.store" "{id}" needs a cache locator» | три `match` |
| `Dispatch\DispatcherFactory::fromConfig(Config, SubmitterInterface, LoggerInterface, ?Closure $queue = null): DispatcherInterface` | `!enabled \|\| none` → `NullDispatcher`; `sync` → `SyncDispatcher`; иначе `$queue()` или `ConfigurationException` «"dispatch" "{mode}" needs a queue dispatcher» | три копии |
| `Collector\Collector::fromConfig(Config, LoggerInterface)` | `collectorDetectLeaks`, `logUrls` | три копии |
| `Throttle\TokenBucket::fromConfig(Config, LoggerInterface)` | `throttleMaxRequestsPerMinute` | три копии |
| `Url\AttributeUrlResolver::fromConfig(Config, AttributeReaderInterface, ?RouteUrlResolverInterface, ?ResolverLocatorInterface, LoggerInterface)` | `maxViaDepth`, `maxViaFanout`, `localeHosts` | семь позиционных аргументов ×3 |
| `Key\KeyFileResponder::fromConfig(Config, KeyProviderInterface)` | `serveKeyFile`, `keyFileMaxAge`, `hosts` | |

Конструкторы всех классов остаются; фабрики — сахар с одним источником текстов ошибок. `IndexNowKit::create()`
переписывается через них, сохраняет сигнатуру (минус `$sitemap`) и проверку несовместимой комбинации «`$submitter` +
`$transport/$debounce/$throttle/$normalizer`»; следствие: `create()` с `dispatch` вне `{sync, none}` и без
`$dispatcher` теперь бросает `ConfigurationException` (раньше молча ставил `SyncDispatcher`). Laravel и бандл
переводят тела binding'ов/сервисов на фабрики уже в волне A (service id и binding'и не меняются); Yii2 — свои
приватные методы.

Уточнения по реализации (волна A):

- Локатор `DebounceStoreFactory::fromConfig` может вернуть не только PSR-16 кэш, но и готовый
  `DebounceStoreInterface` — для фреймворков, чей кэш не PSR-16 (Yii2 отдаёт `YiiCacheDebounceStore`). Так гейт
  «`match ($store`» выполняется и в Yii2 без второго слоя.
- Бандл: `http.client` и `debounce.store` — compile-time факты контейнера (service id), поэтому
  `indexnowkit.transport.real` остаётся своей фабрикой (оборачивает symfony/http-client, проверка через
  `TransportFactory::psr18()`), а `indexnowkit.debounce_store` — compile-time ветвлением на определения;
  `TransportFactory::lazy()` и `DebounceStoreFactory` в бандле не используются. Остальные сервисы
  (`throttle`, `collector`, `url_resolver`, `key_file_responder`, `resolver_locator`, `sitemap_reader`) — фабрики.
- `Url\ArrayResolverLocator(locate:, hint:)`: бандл строит его через статическую `Url\ResolverLocatorFactory::create(ContainerInterface)`
  (замыкание нельзя описать как DI-аргумент); классы `ContainerResolverLocator` трёх адаптеров удалены.
- `Check\DebounceStoreCheck(Config, ?Closure $probe = null, string $default = 'memory')`: третий параметр —
  store адаптера при незаданном `debounce.store` (Laravel/Yii2 `cache`); probe вынесен в `Laravel\Check\CacheStoreProbe`
  и `Yii2\Check\CacheProbe` (invokable), классы `CacheStoreCheck`/`CacheCheck` удалены.
- `CollectorInterface::count()` добавлен (tier «may grow»); `Collector` его уже имел.
- `Config::serveKeyFileFrom(array $raw): bool` (статическая, tier Call): единственный источник правила «явный
  `serve_key_file` побеждает `key_file.enabled`, иначе `true`», с тем же разбором строк, что в `fromArray()`;
  `fromArray()` вызывает её сама. Нужна адаптерному коду, который намеренно не строит `Config`: маршрут ключа Laravel
  регистрируется в `boot()` по сырому массиву (тесты и пакеты биндят свой `Config` после boot), `UrlManagerCheck` Yii2
  читает сырые опции компонента.
- `Sitemap\Console\SitemapRunner` не принимает `Vocabulary` (§1.3): раннер не печатает ни одного слова из него, поэтому
  единственный фреймворк-специфичный текст — аргумент `sitemapUrlOption`.
- `IndexNowKit::create()` с `dispatch` вне `{sync, none}` и без `$dispatcher` бросает `ConfigurationException`
  (следствие `DispatcherFactory::fromConfig()` без `$queue`; раньше молча ставил `SyncDispatcher`) — в CHANGELOG core
  0.4.0 как Changed.
- Laravel `indexnow:sitemap` при `sitemap.enabled: false` печатает `sitemap.enabled is false.` и выходит с `INVALID`
  (раньше игнорировал флаг), как Yii2; бандл команду не регистрирует (§1.2).
- Бандл: `DependencyInjection\TransportFactory::create(?object $client, float $timeout, string $id =
  'indexnowkit.http.client')` — третий параметр `$id` для текста ошибки `TransportFactory::psr18()`; symfony/http-client
  (включая scoped clients) оборачивается в `Psr18Client`, PSR-18 сервис берётся как есть, `null` — discovery.
- Root CI: phpstan на doctrine выполняется только на `highest` (DBAL 4 / ORM 3): DBAL 3 / ORM 2 объявляют сигнатуры в
  docblock'ах, противоречащих DBAL 4 / ORM 3, один код не проходит level 9 на обоих. Условие `if: matrix.package !=
  'doctrine' || matrix.deps == 'highest'` в `ci.yml`; в split-CI doctrine аналогично.
- README core EN/RU: Google-абзац «keep your sitemap for Google» остаётся (§1.2) — это про протокол, не про пакет;
  строка таблицы Family — единственное упоминание пакета.

### 3.2. `Adapter\Services` и `Adapter\ServicesBuilder` (слой 2, волна B)

```php
namespace IndexNowKit\Adapter;

final class ServicesBuilder
{
    public function __construct(Config $config, LoggerInterface $logger = new NullLogger()) {}

    // Каждый узел: готовый объект или Closure(Services): объект. Порядок вызовов не важен.
    public function transport(TransportInterface|Closure $transport): self;
    public function httpClientLocator(Closure $locator): self;        // fn(string $id): object — как адаптер находит http.client
    public function keys(KeyProviderInterface|Closure $keys): self;
    public function normalizer(UrlNormalizerInterface|Closure $normalizer): self;
    public function throttle(ThrottleInterface|Closure $throttle): self;
    public function debounceStore(DebounceStoreInterface|Closure $store): self;   // единственный способ дать не-memory store
    public function client(ClientInterface|Closure $client): self;
    public function submitter(SubmitterInterface|Closure $submitter): self;
    public function events(EventDispatcherInterface $events): self;   // PSR-14 для Submitter
    public function collector(CollectorInterface|Closure $collector): self;
    public function dispatcher(DispatcherInterface|Closure $dispatcher): self;
    public function queueFactory(Closure $factory): self;             // fn(Services): DispatcherInterface для dispatch ∉ {sync, none}
    public function reader(AttributeReaderInterface|Closure $reader): self;    // default RuleRegistry(new AttributeReader())
    public function router(RouteUrlResolverInterface|Closure $router): self;
    public function resolverLocator(ResolverLocatorInterface|Closure $locator): self;
    public function urlResolver(UrlResolverInterface|Closure $resolver): self; // замена AttributeUrlResolver целиком
    public function checks(iterable|Closure $checks): self;           // iterable<CheckInterface> сверх встроенных

    /** @throws ConfigurationException статически известные ошибки: debounceStore-id без debounceStore(), dispatch ∉ {sync,none} без queueFactory()/dispatcher() */
    public function build(): Services;
}

final class Services
{
    public readonly Config $config;
    public readonly LoggerInterface $logger;

    // Ленивые, мемоизированные. Каждый метод = ровно один вызов фабрики §3.1 или конструктора; никакой своей логики.
    public function transport(): TransportInterface;
    public function keys(): KeyProviderInterface;
    public function normalizer(): UrlNormalizerInterface;
    public function throttle(): ThrottleInterface;
    public function debounceStore(): DebounceStoreInterface;
    public function client(): ClientInterface;
    public function submitter(): SubmitterInterface;
    public function collector(): CollectorInterface;
    public function dispatcher(): DispatcherInterface;
    public function reader(): AttributeReaderInterface;
    public function rules(): RuleRegistry;                 // reader, если RuleRegistry; иначе оборачивает: new RuleRegistry($reader)
    public function router(): ?RouteUrlResolverInterface;
    public function resolverLocator(): ?ResolverLocatorInterface;
    public function urlResolver(): UrlResolverInterface;
    public function guardedResolver(): GuardedUrlResolver;
    public function changes(): ObjectChangeHandler;
    public function kit(): IndexNowKit;
    public function keyFileResponder(): KeyFileResponder;
    public function checker(): CheckerInterface;
    public function submitterFactory(): SubmitterFactoryInterface;   // Console\SubmitterFactory над узлами

    /** false, если фасад/коллектор ещё не собирались или коллектор пуст — без сборки. */
    public function hasCollected(): bool;
    /** kit()->flush() в try/catch → error-лог. Никогда не бросает. */
    public function flushIfCollected(): void;
}
```

Инварианты:

- Никакого IO при `build()`. Транспорт — `LazyTransport`, очередь — замыкание, вызываемое при первом обращении.
- Переопределение узла делает зависимые узлы производными (переопределили `transport` — `client()` и `checker()` берут его).
- `Services` — тонкая мемоизация над слоем 1. Тест паритета в core: для каждого публичного метода `Services` есть
  фабрика/конструктор слоя 1, и объект, собранный через `Services`, конфигурационно эквивалентен собранному напрямую
  (сравнение по `Config`-зависимым полям через `FakeTransport`/`ArrayLogger`). Это защита от дрейфа двух слоёв.
- Фича-пакеты не знают о `Services`, и `Services` не знает о них: sitemap строится как
  `SitemapReader::fromConfig($cfg, $services->transport(), $services->logger)`.
- Staging (`TransactionStaging`/`VerifyingStaging`), observer'ы и маршрут key file остаются на стороне адаптера.
- `IndexNowKit::create()` НЕ реализуется поверх `Services` (оставляет свою проверку комбинаций и plain-PHP форму).

Уточнения по реализации (волна B):

- Граф читает правила через `rules()`, не через `reader()`: `urlResolver()`, `guardedResolver()`, `kit()` получают
  `RuleRegistry` (сам reader, если он уже `RuleRegistry`, иначе обёртка), поэтому `rules()->register()` виден всем
  потребителям при любом переданном reader. `reader()` возвращает узел как дан.
- `changes()` — `kit()->changes()` (собственный обработчик фасада), а не второй экземпляр `ObjectChangeHandler`;
  `guardedResolver()` — тот же объект, что `kit()->resolver()`.
- `kit()` всегда получает `transport()` (в отличие от `create()`, где `$transport` `null` при кастомном submitter'е):
  у `Services` транспорт всегда выводим, и sitemap-пакет берёт его оттуда.
- `build()` проверяет статические ошибки самими фабриками: `TransportFactory::lazy()` (id без локатора) и
  `DebounceStoreFactory::fromConfig()` (id без `debounceStore()`) вызываются только ради исключения, их результат
  отбрасывается; проверка `dispatch` — текст `DispatcherFactory` («needs a queue dispatcher…»); при `enabled: false`
  очередь не требуется (как в фабрике). Тип результата замыкания проверяется при первом обращении:
  `LogicException` «ServicesBuilder::<узел>(): the closure must return <тип>, got <тип>».
- `checks()` принимает `iterable` или `Closure(Services): iterable`; замыкания узлов получают `Services` первым
  аргументом. Конструктор `Services` — `@internal`, строится только `ServicesBuilder::build()`; имена узлов —
  константы `Services::TRANSPORT` и т. д.
- Билдер мутабельный (fluent `self` возвращает `$this`), как принято у билдеров; `build()` можно звать один раз на
  описание.
- Тест паритета `ServicesParityTest`: рефлексией перечисляет публичные методы `Services`, для каждого (кроме
  `hasCollected`/`flushIfCollected`) есть двойник, собранный вручную фабриками §3.1; сравнение — рекурсивный экспорт
  свойств (замыкания и показания часов `TokenBucket` заменены плейсхолдерами).
- Yii2: новый публичный `IndexNowComponent::services(): Services`; прежние методы графа (`kit()`, `transport()`,
  `debounceStore()`, …) — делегаты. `routeResolver()` остаётся не-nullable (компонент всегда даёт роутер).
  Переопределения-свойства проходят через `Instance::ensure` внутри замыканий узлов; `flushIfCollected()` —
  `services?->flushIfCollected()`, так что запрос без сбора ничего не строит.

Yii2: `IndexNowComponent` держит `ServicesBuilder` с замыканиями `httpClientLocator` (`App::component() ?? Yii::$container`),
`debounceStore` (через `YiiCacheDebounceStore`: кэш Yii не PSR-16, поэтому переопределяется store, а не локатор кэша),
`queueFactory` (`QueueDispatcher`), `router`, `resolverLocator`, `checks`. Публичные методы компонента становятся
делегатами; свойства-переопределения (`transport`, `debounceStore`, `dispatcher`, `urlResolver`, `logger`, `checks`)
остаются API. Laravel и бандл остаются на слое 1: их binding'и/service id — публичный API, `Services` его сломал бы.

## 4. Общие блоки

### 4.1. Мелкие блоки (волна A)

- `Console\ClassNameResolver(array $namespaces, Closure $accepts, string $expected)`: `resolve(string $class): class-string`;
  тексты `Class "%s" not found.` и `"%s" is not %s.` один раз. `EntityLoader`, `ModelLoader`, `SubjectLoader` делегируют.
- `Check\DebounceStoreCheck(Config $config, ?Closure $probe = null)`: `debouncePerUrl <= 0` → ok «debounce off»;
  `none` → ok «no store»; `memory` → warning «per-process only; set debounce.store to a shared cache» одним текстом;
  иначе `$probe($id)`: адаптер сам пишет/читает тестовый ключ и проверяет тип, возвращает подпись хранилища (ok) или
  бросает (error с текстом исключения). Laravel `CacheStoreCheck` и Yii2 `CacheCheck` становятся замыканием `probe`.
- `Url\ArrayResolverLocator` получает `?Closure $locate = null` (`fn(string $id): ?object`; `null` = провалиться в путь
  «класс без обязательных аргументов») и `?string $hint = null` (что перечислить в ошибке «neither a service nor a
  class»). Три `ContainerResolverLocator` становятся `new ArrayResolverLocator([], locate: fn($id) => $container->has($id)
  ? $container->get($id) : null, hint: 'a service id')`. Нового класса нет.
- `Url\RuleAwareUrlResolverInterface extends UrlResolverInterface` с `resolveRule(object, UrlRule, Event, int $depth = 0,
  bool $ignoreWhen = false): list<ResolvedUrl>` и `explain(object, Event): list<ResolvedUrl>`. `AttributeUrlResolver`
  реализует; `GuardedUrlResolver` (единственное место, три проверки) проверяет интерфейс. Тир Implement с оговоркой
  «до 1.0 может расти».
- `Check\CheckReport::ok/warning/error` — публичные, тир Call.
- `IndexNowKit::submitAll(iterable $subjects, Event $event = Event::Updated): list<Result>` и
  `urlsForAll(iterable $subjects, Event $event = Event::Updated): list<string>`; дедупликация по всему набору
  (документируется: 100 постов одной категории дают один URL категории). `IndexNowManager` и `IndexNowComponent` делегируют.
- `IndexNowKit::collect()` — `count()` коллектора вместо `count(all())`. Сдвоенные docblock объединяются.
- `Result::urlsOf()` удаляется (deprecated с 0.2.0): `ResultTest` строки 32/40/48 переводятся на `retryableUrls()`/
  `urlsWhere()`, `docs/retries-and-queues.md:74` правится.
- `Console\Vocabulary`: без `sitemapUrlOption`; адаптеры передают аргументы по именам.
- phpstan в CI-матрице также на `lowest` (ловит типизацию старых версий фреймворков).

### 4.2. `Hook\ObserverHelper` (волна B)

```php
namespace IndexNowKit\Hook;

final class ObserverHelper
{
    public function __construct(private readonly IndexNowKit $indexNow, private readonly LoggerInterface $logger = new NullLogger()) {}

    /**
     * Никогда не бросает. null = резолв упал (залогировано `indexnow: cannot resolve the URLs of {class}: {error}`),
     * [] = правил нет или ничего не изменилось. Дедуплицированные строки.
     *
     * @param callable(ObjectChangeHandler): list<ResolvedUrl> $resolve
     * @return list<string>|null
     */
    public function guard(object $subject, callable $resolve): ?array;

    /** debug на каждый URL: `indexnow: {source} ({event}) -> {url}` */
    public function logResolved(object $subject, Event $event, array $resolved): void;

    /** collect() в try/catch → error-лог. */
    public function deliver(array $urls): void;

    /** Удаление: резолв до исчезновения строки, доставка после. WeakMap по объекту. */
    public function rememberDeletion(object $subject, array $urls): void;
    public function takeDeletion(object $subject): ?array;
}
```

Без `changeSet()`: вычисление change set и `expected` для верификатора у Laravel (`getRawOriginal()`) и Yii2
(`oldAttributes`, `afterInsert` только по не-null) различается по существу и остаётся в адаптерах. Граница коммита
остаётся в адаптере: `Connection::afterCommit()` (Laravel), `TransactionStaging` + middleware (Doctrine),
`VerifyingStaging` + события соединения (Yii2). Тексты логов удаления: адаптеры либо переходят на тексты хелпера, либо
`docs/operations.md` фиксирует свои.

Уточнения по реализации (волна B):

- `logResolved(array $resolved)` без параметров `$subject`/`Event`: каждый `ResolvedUrl` несёт свой event и источник,
  лишние параметры были бы мёртвыми. `guard()` сам вызывает `logResolved()`; отдельный вызов — для адаптеров, которые
  резолвят вне `guard()`.
- Тексты логов удаления: адаптеры перешли на тексты хелпера (`cannot resolve the URLs of {class}: {error}` и для
  «до удаления»; отдельной строки «before deletion» больше нет). Строки хелпера — в `docs/operations.md` core, раздел
  «ORM hooks».
- Yii2 `IndexNowObserver` держал второй `WeakMap<Connection, true>` (подключения с уже повешенными commit/rollback);
  он заменён на `SplObjectStorage` (подключения живут всё приложение), чтобы гейт §11 «`WeakMap` только в
  `ObserverHelper`» выполнялся буквально.
- `$enabled` (флаг `eloquent.enabled` / `active_record.enabled`) остаётся у адаптера: хелпер не знает о нём.

### 4.3. `Retry\WorkerOutcome` (волна B)

```php
namespace IndexNowKit\Retry;

final readonly class WorkerOutcome
{
    /** @param list<Result> $results */
    public static function of(array $results): self;

    /** @var list<string> */ public array $retryUrls;      // retryable
    /** @var list<string> */ public array $finalUrls;      // failed && !retryable
    /** @var list<string> */ public array $finalReasons;   // уникальные: "api 403", "yandex unprocessable"
    public ?int $retryAfter;                                // максимум Retry-After среди retryable; null = нет

    public function hasRetryable(): bool;
    public function hasFinalFailures(): bool;

    /** Только для фреймворков с задержкой из job'а (Laravel): policy->delayAfter(attempt, retryAfter); null = сдаться. */
    public function delay(RetryPolicy $policy, int $attempt): ?int;

    /** @return array{0: string, 1: array<string, mixed>} шаблон и контекст для PSR-3 */
    public function retryLog(Vocabulary $words, string $jobId, ?int $delay, int $attempt): array;   // 'indexnow: {count} URL(s) of job {id} will be retried{delay} (attempt {attempt})'
    public function gaveUpLog(Vocabulary $words, string $jobId, int $attempt): array;
    public function finalLog(Vocabulary $words, string $jobId, string $checkCommand): array;        // '... rejected permanently ({reasons}); run "{check}"'
}
```

Уточнения по реализации (волна B):

- Лог-методы без `Vocabulary`: в словаре нет ни одного слова, которое им нужно; команда check передаётся строкой
  `$checkCommand` (Laravel `php artisan indexnow:check`, Messenger `bin/console indexnow:check`, Yii2
  `php yii indexnow/check`). Сигнатуры: `retryLog(string $jobId, ?int $delay = null, ?int $attempt = null)`,
  `gaveUpLog(string $jobId, int $attempt)`, `finalLog(string $jobId, string $checkCommand)`. `$attempt` в `retryLog`
  nullable: Messenger не сообщает job'у номер попытки (yii2-queue тоже нет, с yii2 0.5.0 job несёт `attempt` сам); шаблон
  `indexnow: {count} URL(s) of job {id} will be retried{delay}{attempt}` с `{delay}` = ` in {n}s` и `{attempt}` =
  ` (attempt {n})` или пустыми строками. Messenger теперь тоже говорит «job {id}» (было «message {id}»,
  `docs/messenger.md` бандла обновлён).
- `of()` принимает `list<Result>` и хранит результаты для `delay()` (`RetryPolicy::delayAfter()` считает по ним).
- Контекст лог-строк не содержит списков URL (как и раньше в адаптерах): `logging.max_urls` на них не влияет.
- Messenger-хендлер логирует финальные отказы через `finalLog()` (раньше — только строка клиента) и затем, если есть
  retryable, бросает `RecoverableMessageHandlingException` с `retryAfter` (Symfony ≥ 7.2).

Три воркера сводятся к `WorkerOutcome::of($results)` и своим действиям: Laravel `release($o->delay(...))`/`fail()`,
Messenger `throw new RecoverableMessageHandlingException` с `retryAfter`, Yii2 — перепуш нового `SubmitUrlsJob` с
остатком, тем же `id`, `attempt + 1` и `$queue->delay($o->delay($policy, $attempt))` (yii2 0.5.0; до него —
`canRetry()` без задержки, у yii2-queue нет `release($delay)`). `canRetry()` остаётся только для исключений
(`RetryableSubmissionException` кастомного транспорта/сабмиттера). Sync-драйвер игнорирует задержку — документировано.

### 4.4. `Console\Definitions` (волна B)

Декларативные описания опций каждого раннера — имена, короткие флаги, типы, описания, дефолты — в одном месте:
`Definitions::check()`, `::submit()`, `::submitSubjects(Vocabulary)`, `::explain(Vocabulary)`, `::keyGenerate()`;
sitemap-пакет добавляет `Sitemap\Console\Definitions::sitemap()`. Адаптеры строят из них Symfony `configure()`
(бандл, Laravel `$signature` генерируется хелпером `Definitions::laravelSignature()`), Yii2 — `options()`/`optionAliases()`.
Расхождения описаний между тремя адаптерами (сегодня есть) исчезают; тест в core проверяет, что каждое определение
покрывает `SubmitSubjectsOptions`/`SitemapOptions`.

Уточнения по реализации (волна B):

- Модель: `Console\CommandDefinition(description, arguments: list<ArgumentDefinition>, options: list<OptionDefinition>)`
  с методами `argument(name)`, `option(name)`, `applyTo(Symfony\Command)` (аргументы, опции, описание),
  `laravelSignature(string $command): string`, `yiiOptions(): list<string>` (camelCase-свойства контроллера),
  `yiiAliases(): array<string, string>`. `OptionDefinition(name, description, mode: flag|value|optional, shortcut,
  default)` с `property()` (`dry-run` → `dryRun`); `ArgumentDefinition(name, description, required, array)`.
  `laravelSignature()` живёт на `CommandDefinition`, не на `Definitions` (спека называла
  `Definitions::laravelSignature()`): рендер — свойство определения, как `applyTo()`.
- `Definitions::submitSubjects(Vocabulary $words, string $classArgument = 'class')` и `explain(Vocabulary $words,
  string $classArgument = 'class')`: имя аргумента класса — публичный API команды (`ArrayInput(['class' => …])` в
  бандле, `{model}` в artisan), поэтому адаптер передаёт своё; описание аргумента — `<Subject> class (FQCN or short
  name)`. `Definitions::keyGenerate(string $defaultEnvFile = '.env')`: файл по умолчанию печатается в описании
  `--write-env` (бандл `.env.local`, Laravel/Yii2 `.env`). `Sitemap\Console\Definitions::sitemap(string
  $sitemapUrlOption = 'sitemap.url')` — имя опции печатается в описании аргумента (Laravel `indexnow.sitemap.url`).
- Опция с необязательным значением (`--write-env`) в Symfony — `VALUE_OPTIONAL` с дефолтом `false` (как было), в
  artisan — `{--write-env= : …}` плюс `hasParameterOption()` в команде (как было), в Yii — свойство `writeEnv`.
- Описания команд унифицированы: бандл в `#[AsCommand(description:)]` держит тот же текст (атрибут — константа,
  нужен ленивой загрузке), `applyTo()` ставит его же; Laravel берёт `$definition->description` в конструкторе перед
  `parent::__construct()` вместе с `$signature`. Yii2 `options()`/`optionAliases()` строятся из определений;
  описания и дефолты опций в `php yii help indexnow/<action>` — тоже: `IndexNowController::getActionOptionsHelp()`
  подменяет `comment`/`default` из `CommandDefinition` действия (yii2 0.5.0; раньше печатались docblock'и свойств
  контроллера). `sitemap` без пакета — прежнее поведение Yii.
- Тест `DefinitionsTest` в core: сигнатура artisan снапшотом, `applyTo()` через `Symfony\Component\Console\Command`,
  списки Yii, покрытие `SubmitSubjectsOptions` (порядок параметров конструктора = порядок аргументов + опций);
  `Sitemap\Tests\Unit\DefinitionsTest` — то же для `SitemapOptions`.

## 5. Тесты и покрытие

Волна A: перенос sitemap-тестов, `ConfigFactoryTest`, тесты фабрик, регрессии на `key_file.enabld`, паритет `create()`
до/после через `FakeTransport`; phpstan lowest.

Волна B:

- Вместо абстрактных `TestCase`-китов (H03 требует отдельной конфигурации приложения, `runCheck()`-драйверы у трёх
  адаптеров несовместимы) — статические assertion-хелперы: `Testing\KeyFileAssertions::assertKeyFileResponse(int $status,
  array $headers, string $body, string $key, int $maxAge, bool $expectVaryHost)` (парсит директивы `Cache-Control`, а не
  сравнивает строку; `Vary: Host` — только при `hosts` map), `assertNotServed(...)`; `Testing\CheckOutputAssertions::
  assertKeyFileHint(string $output)`, `assertExitCode(...)`. Адаптеры оставляют свои H-тесты, но проверки — из хелперов.
- Coverage: job `coverage` в root `ci.yml` (PHP 8.3, `coverage: pcov`), `phpunit --coverage-clover` для core и sitemap;
  порог фиксируется после первого замера в `tests/coverage-floor.txt`, скрипт сравнения — падение ниже floor = failure.
- Тест паритета `Services` (§3.2), тесты `ObserverHelper`, `WorkerOutcome`, `Definitions`.

## 6. Изменения по адаптерам

| Пакет | Волна A | Волна B |
|---|---|---|
| `doctrine` 0.3.0 / 0.3.1 | `core ^0.4`; типы при необходимости | `core ^0.5` |
| `symfony-bundle` 0.4.0 / 0.5.0 | `ConfigFactory` → `Adapter\ConfigFactory` (auto резолвится в loader); сервисы через фабрики слоя 1; sitemap через `SitemapConfig`/`fromConfig`; `EntityLoader` на `ClassNameResolver`; `ContainerResolverLocator` → `ArrayResolverLocator(locate:)`; allowed без голого `key_file` | `SubmitUrlsHandler` на `WorkerOutcome`; `Definitions`; assertion-хелперы |
| `laravel` 0.5.0 / 0.6.0 | то же слоем 1; `ModelLoader`; `CacheStoreCheck` → `DebounceStoreCheck` с probe; `IndexNowManager` делегирует `submitAll`/`urlsForAll`; sitemap через `fromConfig` | `IndexNowObserver` на `ObserverHelper`; `SubmitUrlsJob` на `WorkerOutcome`; `Definitions` |
| `yii2` 0.2.0 / 0.3.0 | `ConfigFactory`; фабрики; loader; `CacheCheck` → `DebounceStoreCheck`; именованные аргументы `Vocabulary`; `docs/troubleshooting.md` + раздел Debugging в README; sitemap через `fromConfig` | `Services`; observer на `ObserverHelper`; job на `WorkerOutcome`; `Definitions` |
| `sitemap` 0.1.0 / 0.1.1 | новый | `Definitions::sitemap()` |

Публичные API адаптеров (конфиг-ключи, команды, service id, binding'и, свойства компонента) не меняются, кроме:
sitemap-классы в другом пакете (транзитивно установлен), `Vocabulary` без `sitemapUrlOption`, `serve_key_file` deprecated
в пользу `key_file.enabled` (оба работают).

## 7. Документация

- core README (EN/RU): intro и Install перечисляют все адаптеры и sitemap-пакет; таблица Family; «Extension points» +
  фабрики (A), `Adapter\Services` (B).
- `docs/adapters.md`: §2 «20-minute adapter» на фабриках + `ConfigFactory` (A), затем на `ServicesBuilder` +
  `ObserverHelper` (B) — реальный код, проверяемый тестом в core; §14 таблица команд без sitemap (ссылка на пакет);
  §17 `core ^0.4`; §18 таблица с yii2 и колонкой «слой 1 / слой 2»; §20 «Чек-лист DoD адаптера».
- `docs/bc.md`: тиры (`Adapter\*` — Call; `RuleAwareUrlResolverInterface` — Implement, может расти до 1.0; `CheckReport`
  writers — Call); «pin ^0.4.0»; таблица deprecations (`serve_key_file`, удалённый `urlsOf`, `sitemap()`).
- `docs/configuration.md`: новые ключи; `docs/operations.md`: тексты новых лог-строк.
- Sitemap-пакет: README (EN/RU по шаблону 90), SECURITY.md, `docs/bc.md`, CHANGELOG, `docs/adapters.md` («как подключить
  команду в свой адаптер»).
- Адаптеры: README (Sitemaps через пакет; Debugging и `docs/troubleshooting.md` у yii2), `docs/extending.md`.
- Спека: 02 (таблица API, схема конфига с `key_file.*`, `debounce.store`, `http.client`, блок `sitemap` как пакет; убрать
  `MapKeyProvider`; throttle в `Client`), 10 (пакет sitemap, `Adapter\*`, PHP 8.2–8.5), 12/13/15 (что переехало в core),
  90 (пакет в семействе), 91 (статус), README индекса (16).
- Changelog'и всех пакетов с миграцией; сводный `php/CHANGELOG.md` — новая волна.

## 8. Порядок работ и релизы

Каждая фаза заканчивается зелёным `bin/ci` по всем пакетам и коммитом (conventional commits, сообщение через файл).

### Волна A

1. **Инструментарий.** `php/bin/tag`, `php/bin/packagist-wait`, `php/bin/release-notes` (эквиваленты scratchpad-скриптов)
   в репо; `sitemap` в `tools-push-splits.sh`, `.github/workflows/split.yml` (`SPLIT_SSH_KEY_SITEMAP`), матрицу root
   `ci.yml` (включая lowest 8.2), `php/README.md` (таблица семейства, дерево), карту `release-notes`. Репо
   `indexnowkit/php-sitemap`, deploy-key, Packagist-регистрация — после первого push сплита.
2. **Sitemap.** `packages/sitemap` (§1.3–1.4), перенос классов/тестов/роутера, core без следов (§1.2, гейт grep),
   адаптеры на `SitemapConfig`/`fromConfig`/`require`.
3. **Config + ConfigFactory.** §2; три `ConfigFactory` адаптеров; исправление голых ключей; регрессии.
4. **Фабрики + мелкие блоки.** §3.1, §4.1; `create()` через фабрики; Laravel/бандл/Yii2 на фабрики; phpstan lowest.
5. **Документация, спека, changelog'и.**
6. **Релиз.** `core@0.4.0` → `packagist-wait` → `sitemap@0.1.0` → `doctrine@0.3.0` → `symfony-bundle@0.4.0` →
   `laravel@0.5.0` → `yii2@0.2.0`; GitHub releases из changelog; проверка CI сплитов.

### Волна B

7. `Services`/`ServicesBuilder` + тест паритета; Yii2 на `Services`.
8. `ObserverHelper`, `WorkerOutcome`, `Definitions`; адаптеры переходят.
9. Assertion-хелперы, coverage-гейт, документация, changelog'и.
10. Релиз `core@0.5.0` и адаптеров.

Ветка `main` монорепо; сплиты публикуют `main`, теги — только в фазах релиза. Между фазами адаптеры в сплитах живут на
`core dev-main` (minimum-stability dev уже стоит).

## 9. Совместимость и миграция

### core 0.3 → 0.4

| Изменение | Тип | Миграция |
|---|---|---|
| sitemap вынесен: `IndexNowKit::sitemap()`, `$sitemap` в `create()`/конструкторе, `Console\SitemapRunner`/`SitemapOptions`, `Check\SitemapSpoolCheck`, `Sitemap\*` | breaking | `composer require indexnowkit/sitemap`; `$kit->sitemap()` → `SitemapReader::fromConfig(SitemapConfig::fromArray($block), $kit->transport ?? TransportFactory::lazy($kit->config), $logger)`; `Console\SitemapRunner` → `Sitemap\Console\SitemapRunner` |
| `Vocabulary::$sitemapUrlOption` удалён | breaking | аргумент `SitemapRunner`; вызовы `Vocabulary` только по именам |
| `Result::urlsOf()` удалён | breaking (deprecated с 0.2.0) | `retryableUrls()` / `urlsWhere()` |
| `Config`: `keyFileMaxAge`, `debounceStore`, `httpClient`; `key_file.enabled` в `fromArray`/`with`/`fromEnv`; `keyFileHeaders()` | additive | адаптеры убирают ключи из allowed-списков |
| `Adapter\ConfigFactory`, фабрики §3.1, `Console\ClassNameResolver`, `Check\DebounceStoreCheck`, `ArrayResolverLocator(locate:, hint:)`, `Url\RuleAwareUrlResolverInterface`, `CheckReport` writers public, `IndexNowKit::submitAll/urlsForAll` | additive | — |

### core 0.4 → 0.5

| Изменение | Тип | Миграция |
|---|---|---|
| `Adapter\ServicesBuilder`/`Services`, `Hook\ObserverHelper`, `Retry\WorkerOutcome`, `Console\Definitions` + `CommandDefinition`/`ArgumentDefinition`/`OptionDefinition`, `Testing\KeyFileAssertions`/`CheckOutputAssertions`, `Sitemap\Console\Definitions` | additive | — |
| sitemap 0.1.1 требует core ^0.5 (модель `CommandDefinition`); адаптеры требуют core ^0.5 и sitemap ^0.1.1 | зависимости | обновлять адаптер |
| Тексты логов адаптеров: observer'ы — строки `ObserverHelper` (нет отдельной «before deletion»); воркеры — строки `WorkerOutcome` (Messenger «job {id}» вместо «message {id}», Yii2 «will be retried» вместо «were not accepted…», Laravel с `(attempt N)`) | тексты логов (не API) | алерты по уровням, не по тексту (bc.md) |
| Описания команд унифицированы (`submit-entity`/`submit-model`, `explain`, аргумент класса); имена аргументов/опций, service id, `ArrayInput`-ключи не менялись; `SubmitEntityCommand`/`ExplainCommand` бандла и `SubmitModelCommand`/`ExplainCommand` Laravel получают `Vocabulary` в конструктор | help-тексты; конструкторы команд | только при ручном инстанцировании команд |
| Yii2: публичный `IndexNowComponent::services()` | additive | — |

Breaking в core нет.

## 10. Риски и контрольные вопросы

- **PSR-4 при частичном обновлении.** core 0.3 + sitemap 0.1 в одном vendor невозможны (`require ^0.4`); при `composer
  update` с обновлением только core до 0.4 адаптер 0.3.x всё ещё импортирует `Console\SitemapRunner` — поэтому адаптеры
  волны A требуют `core ^0.4` и релизятся сразу после core, а сводный changelog говорит «обновляйте адаптер, не core».
- **Бандл и `auto`.** Резолв `dispatch: auto` остаётся на компиляции; `ConfigFactory` получает уже конкретный режим.
  Тест loader'а с Messenger и без.
- **Слияние defaults.** Списки в defaults запрещены на уровне конструктора; известные блоки сливаются по второму уровню;
  тест на `engines`/`hosts` из raw без искажений.
- **`serve_key_file` приоритет.** Явный `serve_key_file` побеждает `key_file.enabled` — как сегодня; тест.
- **Laravel binding'и остаются API.** Laravel не переводится на `Services` (иначе `$app->bind(TransportInterface::class)`
  перестанет влиять на граф). Только слой 1.
- **Два слоя дрейфуют.** Тест паритета §3.2 — обязательная часть волны B, не «потом».
- **WorkerOutcome и yii2-queue.** Без задержки из job'а: поведение как сейчас, документировано.
- **Coverage-гейт.** Порог ставится после замера; изменение порога — отдельный коммит с обоснованием.
- **Размеры файлов** — ориентир, не гейт: component/provider/loader ≈ 250–300 строк, observer'ы ≈ 150. Гейты — grep на
  удалённые дубли (§11).
- **Сигнатуры в спеке — контракт для исполнителя.** Отклонение фиксируется в спеке тем же коммитом.

## 11. Definition of Done

Волна A:

- `bin/ci` зелёный для core, sitemap, doctrine, symfony-bundle, laravel, yii2 на всех флейворах; сплит-CI зелёный, включая
  новый `php-sitemap`.
- Гейт §1.2: `grep -ril sitemap packages/core --exclude-dir=vendor --exclude-dir=.phpunit.cache
  --exclude=composer.monorepo.json` → только `CHANGELOG.md`, `README.md`, `README.ru.md`, `composer.json`, `docs/bc.md`,
  `docs/adapters.md`.
- Гейты дублей (grep по `packages/{symfony-bundle,laravel,yii2}/src` пуст): `serve_key_file` — `grep -rn serve_key_file
  packages/{laravel,yii2}/src` пуст, в бандле допускается только `IndexNowKitConfiguration.php` (deprecated-узел дерева —
  конфигурация, не логика); правило «явный `serve_key_file` побеждает `key_file.enabled`» живёт в одном месте —
  `Config::serveKeyFileFrom(array $raw): bool`, которую вызывает `fromArray()` и адаптерный код до построения `Config`
  (маршрут ключа Laravel в boot, `UrlManagerCheck` Yii2); `is not a PSR-18 client`; `needs a cache`/`match ($store`;
  `Class "%s" not found`; `intOf(`; `new SitemapReader(`; `'key_file'` и `'sitemap'` как элементы allowed-списков.
- Регрессионный тест `key_file.enabld` в каждом адаптере.
- Семь пакетов опубликованы, `indexnowkit/sitemap` на Packagist, changelog'и с миграцией, спека синхронизирована,
  `php/bin/{tag,packagist-wait,release-notes}` в репо.

Волна B:

- Гейты дублей: `WeakMap` в observer'ах адаптеров (только через `ObserverHelper`); `retryableUrls()`/`isRetryable` в
  воркерах адаптеров; `->addOption(` с литералами описаний вне `Definitions`. Проверка: `grep -rn "WeakMap\|retryableUrls\|isRetryable\|addOption(\|addArgument(\|protected \$signature" php/packages/{laravel,yii2,symfony-bundle}/src` пуст.
- Тест паритета `Services` зелёный (`ServicesParityTest`); coverage-floor файлы в репо
  (`packages/{core,sitemap}/tests/coverage-floor.txt`, job `coverage` в root `ci.yml`, `bin/coverage-floor`);
  `docs/adapters.md` §2 покрыт тестом в core (`TwentyMinuteAdapterTest`, на `ServicesBuilder` + `ObserverHelper`).
- Повторный аудит по линзам: API 9, авторинг 9, DX пользователя 9, тесты 9, документация 9; протокол/надёжность/безопасность
  9 без регрессий.
