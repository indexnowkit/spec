# 19. Волна M: остатки «велосипедов» после волны L и ревизия PSR по всему семейству

Статус: **реализовано 2026-09-07** (волна M, девять коммитов по пакетам поверх 02c667c, до пуша; решения §6 — по рекомендациям:
1 да, 2 (a), 3 да, 4 (a), 5 да, 6 нет, 7 да, 8 нет). Отклонения от §4, записанные при реализации:
(1) §4.10 `about` Laravel — не `notInstalledMessage()`, а `checkLine([])` без слова-фичи (`not installed (composer require …)`):
два теста `tests/Feature/*NotInstalledTest` проверяют эту подстроку, сценарии не тронуты;
(2) §4.4 Laravel `ChecksTest` и Yii2 `ChecksTest` конструировали удалённые `RouterCheck` — переписаны на `Check\LocalesCheck` (те же
проверки; Yii2 — при пустом списке и без просящих классов строки нет, раньше был ok, как и записано в §4.4);
(3) §4.6 `RouteOrigin::expand()` — флаг `bool &$warned = false` вместо `?bool` (phpstan: свойство `bool` адаптера); Laravel оставляет
свой текст `Cannot generate route "%s": no route has that name.` (роутер возвращает null, не бросает — `generationFailed()` не применим);
(4) §4.3 маркер `AbstractSubjectLoader` — `is_a($class, $marker, true)`, не `is_subclass_of` (класс-маркер сам проходит);
текст guard — тот же, что у `ClassNameResolver`, один на семейство; `findOne()`/`findMany()` без `$how`;
(5) §2.9 `YiiCacheDebounceStore` → `Psr16DebounceStore`: `filterRecent(…, 0)` больше не возвращает `[]` (окно 0 — дело сабмиттера,
стор его не спрашивает) — тест переехал с этой поправкой; (6) §4.10 Yii3 `ObserverProvider::set(…, ?LoggerInterface)` — логгер
живёт после `reset()` (`resetLogger()` для тестов), иначе ветка недостижима; (7) бандл: закрытие `LocalesCheck` над метаданными —
отдельный `Check\MappedClasses` (замыкание нельзя описать в DI без класса); `HistoryServices` бандла тоже держал три `in_array` —
убраны; (8) `SubmitUrlsMessage::newId()`, `SubmitUrlsJob::newId()` ×2 оставлены делегатами на `BatchingDispatcher::newJobId()`.
Не сделано: ничего из §4. Найдено до волны и не чинилось (не в её границах): `bin/taint core` — `TaintedHtml` в
`tests/Taint/entrypoints.php:37` (`echo` тела ключ-файла в харнесе; воспроизводится на 02c667c); `bin/ci symfony-bundle symfony64` и
`lowest` — `ignore.unmatchedLine` в `tests/App/Controller/ArticleController.php:50` и `tests/Functional/MessengerDispatchTest.php:47`
(phpstan level 6 по тестам на другом vendor). Аудит — 2026-09-07, тем же днём.

## 0. Цель и границы

Два вопроса. (1) Где в семействе после волны L (спека 18) два и более адаптера — или адаптер и пакет — держат
похожие копии кода, либо повторяют то, что уже даёт core, пакет, фреймворк или готовая PSR-реализация. (2) Где семейство
использует PSR не по букве стандарта, не полностью, или не там, где стандарт дал бы адаптеру или пользователю выигрыш.

Границы: пакеты `php/packages/*` на состоянии `main` 9230ce5 (волна L закоммичена, не запушена, не выпущена; все версии
Unreleased: core 0.13.0, console 0.5.0, testing 0.3.2, sitemap 0.8.0, verify 0.4.0, history 0.4.0, doctrine 0.9.0,
symfony-bundle 0.15.0, laravel 0.15.0, yii2 0.14.0, yii3 0.1.0). Битрикс и фаза B `bin/indexnow` (спека 18 §8) — не здесь.

Пороги, как в спеке 18 §1: ≥ 70 % сходства строк — копия, переносить; 50–70 % — смотреть, что именно совпадает (обычно
обвязка вокруг одного вызова core — тогда хелпер в core или пакете); < 50 % — фреймворковая обёртка, оставить, если нет
другой причины (одинаковый текст, одинаковая константа, знание о пакетах в трёх местах).

## 1. Факты

### 1.1. Замер сходства (2026-09-07)

`difflib.SequenceMatcher.ratio()` по строкам после удаления пустых строк и комментариев (в спеке 18 §0 комментарии не
удалялись — там числа на 3–5 пунктов выше; порядок находок тот же). В скобках — длина файла в строках.

| Роль | Файлы | Сходство | Вердикт |
|---|---|---|---|
| ORM-сэмплер `--sample-class` | bundle `Check/EntitySampler` (38), laravel `Check/ModelSampler` (38), yii2 `Check/RecordSampler` (38), yii3 `Check/RecordSampler` (38) | **72.7 % попарно, yii2~yii3 95.5 %**; различаются только docblock и имя свойства | копия ×4 → §2.2 |
| Загрузчик субъектов `SubjectLoaderInterface` | bundle `Command/EntityLoader` (67), laravel `Console/ModelLoader` (100), yii2 `ActiveRecord/ActiveRecordLoader` (75), yii3 `ActiveRecord/ActiveRecordLoader` (97) | 49–63 %, yii2~yii3 **70.4 %** | общий каркас → §2.3 |
| Проба кеша | bundle `Check/CacheProbe` (26), laravel `Check/CacheStoreProbe` (27), yii2 `Check/CacheProbe` (29), yii3 `Check/CacheProbe` (45) | 30–47 % | оставить; унифицировать ключ и запись → §2.4 |
| Проверка локалей `router.locales` | bundle `Check/LocalesCheck` (81), laravel `Check/RouterCheck` (77), yii2 `Check/RouterCheck` (62) | laravel~yii2 50.5 %, с бандлом 30–35 %; **одна и та же проверка с тремя разными текстами** | в core → §2.4 |
| Проверка ключ-файла/роутера | yii3 `Check/RouterCheck` (98), yii2 `Check/UrlManagerCheck` (63) | 48 % | оставить (разные роутеры) |
| ORM-check | yii2 `Check/ActiveRecordCheck` (29), yii3 (47), laravel `Check/EloquentCheck` (26) | 38–54 % | оставить (две строки, разные существительные) |
| Проверка очереди | laravel `Check/QueueCheck` (45), yii2 `Check/QueueCheck` (44), yii3 `Check/DispatchCheck` (44) | 41–50 %; общая — только строка `dispatch "%s": URLs are %s` | оставить; текст в core → §2.12 |
| Диспетчер очереди | bundle `Messenger/MessengerDispatcher` (50), laravel `Queue/QueueDispatcher` (51), yii2 `Queue/QueueDispatcher` (53) | 49–67 %; общий — цикл `array_chunk` + `newId()` + try/catch + два лога | каркас в core → §2.5 |
| Джоба очереди | bundle `Messenger/SubmitUrlsHandler` (64), laravel `Queue/SubmitUrlsJob` (104), yii2 `Queue/SubmitUrlsJob` (95) | 20–32 % | честно разные → оставить |
| Ключ-файл | bundle `Controller/KeyFileController` (31) ~ laravel `Http/KeyFileController` (32) **84 %**; yii2 (46); yii3 `Http/KeyFileHandler` (38, над core) | | оставить с причиной → §2.6 |
| `ConfigFactory` адаптера | bundle (80), laravel (89), yii2 (101), yii3 (95) | 39–53 % | задуманная форма; 3 копии знания о пакетах → §2.7 |
| `*RouteUrlResolver` | bundle (85), laravel (138), yii2 (144), yii3 (113) | 36–45 %; общие `locales()` ×3 дословно, `rebase()` laravel=yii2 дословно, четыре текста | хелпер в core → §2.8 |
| `SubjectReader` | laravel (63), yii2 (49), yii3 (52) | 43–46 %, yii2~yii3 70.4 % (общий AR-API) | оставить |
| Observer ORM | laravel (241), yii2 (340), yii3 (509) | 14–33 % | оставить |
| `Wiring` | yii2 (248), yii3 (456) | 12.7 % | оставить |
| `{History,Sitemap,Verify}Services` адаптера | bundle vs laravel | 25–36 % | оставить |
| Логгер-мост | yii2 `Log/YiiLogger` (65), yii3 `Log/CategoryLogger` (32) | 35 % | разные задачи → §2.9 |
| `ConfigSourceInterface` | bundle `ConsoleConfigSource` (49), yii3 `Console/ConfigSource` (38) | 57 % | задуманная форма |
| Мост событий | laravel `Event/EventDispatcherBridge` (25), yii2 `Event/ResultDispatcher` (27), yii3 `Event/ObservedDispatcher` (64) | 33–58 % | разные задачи → §2.9 |
| Composition root | bundle `IndexNowKitLoader` (592), laravel `IndexNowKitServiceProvider` (610), yii2 `IndexNowComponent` (668), yii3 `IndexNow` (544) | 3–12 %, yii2~yii3 39 % | оставить |
| Консоль без symfony-классов | laravel `src/Console/*` — **596 строк, 14 файлов**; yii2 `src/Console/*` — 598 строк (контроллер 444, `ControllerOutput` 27, `HistoryAction` 86, `SitemapAction` 41) | laravel-обёртки против классов console 33–50 %: те же тела, другой родитель | Laravel → §2.1; Yii2 оставить → §2.10 |

Сверх списка спеки 18: `Laravel\Console\{Sitemap,History,Status}NotInstalledCommand` (36 строк ×3, 80–85 % между собой)
против `Console\Command\*NotInstalledCommand` (13 строк ×3 над абстрактным) — уходят вместе с §2.1.

### 1.2. Повторяющиеся тексты и константы (grep по `src/` всех пакетов)

| Что | Где | Сколько |
|---|---|---|
| `in_array($store, [DebounceStoreFactory::MEMORY, DebounceStoreFactory::NONE], true)` — «стор общий или процессный» | bundle `IndexNowKitLoader:265` и `:508` (**литералы `'memory'`, `'none'`**), laravel `IndexNowKitServiceProvider:234`, `History/HistoryServices:154`, yii2 `Wiring:95`, `Console/HistoryAction:66`, yii3 `Wiring:244`, `:321`, history `Adapter/HistoryServices:102` | **9** (спека 16 запрещала копировать этот `match`) |
| `bin2hex(random_bytes(6))` — id джобы | bundle `SubmitUrlsMessage:18`, laravel `SubmitUrlsJob:47`, yii2 `SubmitUrlsJob:48` | 3 |
| `Cannot generate route "%s": %s` | четыре `*RouteUrlResolver` | 4 |
| `$this->config->baseUrlFor($host) ?? 'https://' . $host` | четыре `*RouteUrlResolver` | 4 |
| `No request to take the host from: set base_url …` | yii2, yii3 `YiiRouteUrlResolver` | 2 |
| `"%s" is not an X (it does not extend/implement %s): the command loads … by id through …` | laravel `ModelLoader:80`, yii2 `ActiveRecordLoader:70`, yii3 `ActiveRecordLoader:92` | 3 |
| `not installed (composer require indexnowkit/…)` собранное руками вместо `OptionalPackage::notInstalledMessage()` | laravel `IndexNowKitServiceProvider:463`, `:471` (секция `php artisan about`) | 2 (текст отличается от строки стаба команды) |
| Описания стабов `… (needs indexnowkit/…, which is not installed)` | laravel `Console/*NotInstalledCommand:19` ×3 | 3 |
| `sitemap.enabled is false.` + `ExitCode::INVALID` — гейт, который `SitemapRunner(enabled:)` уже делает сам | laravel `Console/SitemapCommand:31`, yii2 `Console/SitemapAction:33` | 2 |
| Числа `1000` / `50` / `32` вместо `SubmitSubjectsOptions::DEFAULT_LIMIT`, `HistoryCommand::DEFAULT_LIMIT`, `KeyGenerateCommand::DEFAULT_LENGTH` | laravel `SubmitModelCommand:33`, `HistoryCommand:37`, `KeyGenerateCommand:32`; yii2 `IndexNowController:319`, `:355` | 5 |
| Ключ пробы кеша `indexnowkit:check` (с `:`, зарезервированным PSR-16) против `indexnowkit_check` | laravel `CacheStoreProbe:22`, yii2 `CacheProbe:25` против bundle `CacheProbe:22`, yii3 `CacheProbe::KEY` | 2 против 2 |
| Описание debounce-стора для `status` (`<id> (<Class>)`, `memory`, `none`) | laravel `History/HistoryServices:154`, yii2 `Console/HistoryAction:60`, yii3 `Wiring:240` | 3 |

Чего **нет** (проверено): в `sitemap`, `verify`, `history` — своих текстов «not installed», своих разборов `debounce.store`
кроме одного `in_array` выше; в `src/` — файлов с двумя классами; `error_log` — нигде; `trigger_error` — один
(yii3 `ActiveRecord/ObserverProvider:31`, статический контекст без логгера).

### 1.3. Инвентарь PSR (`use Psr\…` в `src/`, 2026-09-07)

| Интерфейс | Употреблений | Где |
|---|---|---|
| PSR-3 `LoggerInterface` / `NullLogger` / `AbstractLogger` / `LogLevel` | 50 / 39 / 3 / 1 | везде; `AbstractLogger` — yii2 `YiiLogger`, yii3 `CategoryLogger`, core `Testing\ArrayLogger` |
| PSR-16 `CacheInterface` / `InvalidArgumentException` | 23 / 1 | core, verify, history, адаптеры; исключение — yii2 `Cache\InvalidKey` |
| PSR-20 `ClockInterface` | 20 | все узлы графа, history; **не** в sitemap (§3, PSR-20) |
| PSR-14 `EventDispatcherInterface` / `StoppableEventInterface` | 14 / 1 | core (эмиттер), три моста адаптеров; `Stoppable` — yii3 `ObservedDispatcher` |
| PSR-11 `ContainerInterface` | 5 | bundle `Url/ResolverLocatorFactory`, `EventListener/FlushListener`; yii3 `IndexNow`, `Wiring`, `Check/CacheProbe` |
| PSR-7 / PSR-17 / PSR-18 | 3+2+1 / 2+1+1 / 3+1 | только core `Http/{Psr18Transport,TransportFactory}`, `Key/KeyFileRequestHandler`; yii3 `Http/KeyFileHandler` (над core) |
| PSR-15 `RequestHandlerInterface` / `MiddlewareInterface` | 2 / 2 | core `KeyFileRequestHandler`, yii3 `KeyFileHandler` |
| PSR-6 `CacheItemPoolInterface` | 1 | bundle `Check/CacheProbe` (только для подписи пула) |

Фреймворки (по `vendor/`): `Illuminate\Contracts\Cache\Repository extends Psr\SimpleCache\CacheInterface`
(`laravel/vendor/laravel/framework/src/Illuminate/Contracts/Cache/Repository.php:8`); `Illuminate\Log\LogManager implements LoggerInterface`;
`Illuminate\Events\Dispatcher` — **не** PSR-14 (ни одного `Psr\EventDispatcher` в `Illuminate/Events/*`); `yii\di\Container extends Component`
(`yii2/vendor/yiisoft/yii2/di/Container.php:106`) — **не** PSR-11; `yii\caching\Cache implements yii\caching\CacheInterface` — **не** PSR-16
(в `yiisoft/yii2` нет `Psr\SimpleCache`); Yii3 — PSR-11/14/15/16 нативно.

## 2. Находки по гипотезам

### 2.1. Laravel регистрирует классы команд `indexnowkit/console` напрямую — ПОДТВЕРЖДЕНО (главная находка)

Спека 18 §9 утверждала «artisan-команды обязаны наследовать `Illuminate\Console\Command`». Это неверно:

- `laravel/vendor/laravel/framework/src/Illuminate/Console/Application.php` (laravel/framework v13.30.1, symfony/console v7.4.18):
  `add(SymfonyCommand $command)` :218, `addCommand(SymfonyCommand|callable $command)` :229 — `setLaravel()` зовётся только
  `if ($command instanceof Command)` (:231, Illuminate-класс), остальное уходит в `parent::addCommand()`; `resolve($command)` :255 —
  для строки-класса, наследующего `Symfony\...\Command` **с `#[AsCommand]`**, кладёт имя в `commandMap` и ничего не строит (:257–268,
  ленивая загрузка через `ContainerCommandLoader::get()` → `$this->container->get($class)`), без атрибута — `$this->laravel->make($command)` :275
  (жадно, при каждом старте artisan); `resolveCommands()` :284. `Illuminate\Support\ServiceProvider::commands()` :474–481 —
  `Artisan::starting(fn ($artisan) => $artisan->resolveCommands($commands))`, то есть `$this->commands([...])` принимает любой класс.
- Запуск: `Kernel::call()` :439–448 → `Application::call()` :157–168 → `run($input, $outputBuffer ?: new BufferedOutput)` — команда получает
  `OutputInterface` как есть. `Illuminate\Console\Command::run()` :250–254 оборачивает вывод в `OutputStyle` только для Illuminate-команд.
- Тесты: `Illuminate\Testing\PendingCommand::run()` :445–452 передаёт **мок `OutputStyle` третьим аргументом `Kernel::call()`**, а ожидания
  `expectsOutputToContain` навешаны на мок `BufferedOutput[doWrite]` под ним (:587–623). Symfony-команда пишет в `new SymfonyStyle($input, $output)`
  поверх этого мока → `doWrite` вызывается → ожидания срабатывают. Из 16 файлов `tests/Feature/*` с командами 12 зовут `Kernel::call(..., BufferedOutput)`
  напрямую (`artisanCall()` в `CommandsTest:264`), 4 — `$this->artisan()` с `expectsOutputToContain`/`assertExitCode` (`DisabledTest:30`,
  `InvalidConfigTest:43`, `DebounceCacheStoreTest:34`, `EventsAndAboutTest:29`). Оба пути работают с голыми Symfony-командами.

Что теряется: `$this->components` (view-компоненты Laravel), `Isolatable`, `PromptsForMissingInput` — ни одна команда семейства их не
использует; `OutputStyle` Laravel — это `SymfonyStyle`; `$this->option()` ≡ `$input->getOption()`; `signature` ≡ `Definitions::applyTo()`.
Планировщик (`$schedule->command('indexnow:sitemap')`) и `Artisan::call()` работают по имени — не затронуты.

Два узких места, оба решаются без изменения сценариев тестов:

1. **Имя аргумента.** Laravel зовёт аргумент класса `model` (`Definitions::submitSubjects($words, 'model')`, `explain(…, 'model')`), console —
   `class`. `Kernel::call('indexnow:submit-model', ['model' => Post::class])` (`CommandsTest:152–200`, 10 вызовов) и такой же вызов у
   пользователей сломался бы. Решение: `Console\Command\SubmitSubjectsCommand` и `ExplainCommand` получают третий именованный аргумент
   конструктора `string $classArgument = 'class'` (console 0.5.0 Unreleased, аддитивно); Laravel передаёт `'model'`. Позиционный CLI не
   меняется в любом случае.
2. **Ленивость.** У `SubmitSubjectsCommand` нет `#[AsCommand]` (имя из `Vocabulary`), значит Laravel построит её жадно при каждом `php artisan`
   вместе с `SubmitSubjectsRunner` и графом. Решение: регистрировать её через `Symfony\Component\Console\Command\LazyCommand`
   (`laravel/vendor/symfony/console/Command/LazyCommand.php`; `Application::addCommand()` понимает его — `Application.php:351`, `:585`):
   `Artisan::starting(fn (Artisan $artisan) => $artisan->addCommand(new LazyCommand('indexnow:submit-model', [], $definition->description, false,
   static fn () => $app->make(SubmitSubjectsCommand::class))))`. Остальные восемь — через `$this->commands([...])` по классам с `#[AsCommand]`
   (ленивая карта). `php artisan list` строит раннеры, как в бандле и Yii3 (`LazyCommand` Symfony ведёт себя так же); IO при этом нет
   (`LazyTransport`, `Config` строится через `ConfigFactory::create()`, который не бросает).

Что остаётся в `laravel/src/Console/`: `ModelLoader` (100) и новый `ConfigSource` (~35, реализация `Console\ConfigSourceInterface`
над `config('indexnow')`, `ConfigFactory::build()` с тремя предикатами и `PACKAGE_BLOCKS`). Уходит **~460 строк** из 596 и
`Definitions::laravelSignature()` из console (единственный потребитель — Laravel; §4.2). Сэмплер — через `SampleOptions::$sampler`, как в Yii3.

### 2.2. Сэмплер `--sample-class` — ПОДТВЕРЖДЕНО: один класс в `indexnowkit/console`

Четыре файла по 38 строк, различаются docblock и именем свойства (`$entities`/`$models`/`$records`); yii2 и yii3 — 95.5 %.
Класс зависит от `Console\SubjectLoaderInterface` (живёт в console, не в core) и `IndexNowKit` → место — **console**:
`Console\SubjectSampler(SubjectLoaderInterface $subjects, IndexNowKit $indexNow)`, `__invoke(string $class, ?string $id): list<string>`,
`PER_CLASS = 3`. Адаптер отдаёт только загрузчик. −152 +38 строк. Ни один из четырёх классов не назван в `bc.md` своего адаптера.

### 2.3. Загрузчики субъектов — ПОДТВЕРЖДЕНО ЧАСТИЧНО: общий каркас, разные запросы

Общее у четырёх (49–70 %): поле `ClassNameResolver`, `resolveClass()` с проверкой класса-маркера, цикл `byIds()` (found/missing),
`all()` с `max(1, $limit)`, текст `"%s" is not an X (…): the command loads … by id through …` ×3. Различается то, что и должно:
предикат и класс-маркер (`Model`, `yii\db\ActiveRecord`, `ActiveRecordInterface`, «управляемая Doctrine сущность»), `findOne`
(`find()`, `findOne()`, `query()->findByPk()`, `$repository->find()`), `findMany` (`limit()->get()`, `find()->limit()->all()`, `batch()`
у yii3, `findBy([], null, $limit)`), `withTrashed()` для `Event::Deleted` у Laravel. Каркас: `Console\AbstractSubjectLoader` (§4.3) —
конструктор `(list<string> $namespaces, string $marker, string $noun)`, абстрактные `findOne()` и `findMany()`, шаблонные `resolveClass()`,
`byIds()`, `all()`. Оценка: 339 → ~160 строк в адаптерах + ~60 в console (−120). Бандл держит `Command\EntityLoader` в «Public classes»
`bc.md` — класс остаётся, только наследует; конструктор не меняется. Ценность ниже, чем у §2.2 (запросы — 60 % каждого файла), риск низкий
(оба ORM-конформанса и `Console/*Test` гоняют команды). `[решение]` §6.3.

### 2.4. Пробы и проверки — ОТВЕРГНУТО для `CacheProbe`/`QueueCheck`/ORM-check, ПОДТВЕРЖДЕНО для локалей

- **`CacheProbe` ×4** — каждая говорит со своим кешем (пул Symfony + PSR-16 view, `Cache\Factory::store()`, компонент Yii2, PSR-16 из контейнера
  Yii3) и уже является замыканием-портом `DebounceStoreCheck::$probe`. Общего кода нет. Две неоднородности: ключ (`indexnowkit:check` в Laravel
  и Yii2 — `:` зарезервирован PSR-16) и действие (bundle/laravel — `get`, yii2/yii3 — `set`; docblock `DebounceStoreCheck` обещает «writes and reads»).
  Исправление: константа `DebounceStoreCheck::PROBE_KEY = 'indexnowkit_check'` в core, `set()` во всех четырёх (4 строки).
- **Проверка локалей** — bundle `LocalesCheck`, laravel `RouterCheck`, yii2 `RouterCheck`: один код `router.locales`, один вопрос
  («список пуст, а правило просит `locales: 'all'`»), три текста, три источника классов (метаданные Doctrine / `--sample-class` /
  `active_record.models`), у бандла нет ok-строки, у Laravel/Yii2 есть. Принцип семейства «тексты — в core» нарушен. Дизайн — `Check\LocalesCheck`
  в core (§4.4): список локалей, `AttributeReaderInterface`, замыкание «какие классы смотреть», имя опции для текста, параметр локали для ok-строки.
  220 → ~70 в core + 3 × ~15 в адаптерах (−100). Laravel `bc.md` называет `Check\RouterCheck` стабильным именем → `[решение]` §6.4.
- **`QueueCheck` laravel/yii2, `DispatchCheck` yii3** — очередные ветки честно фреймворковые; общая одна строка `dispatch "%s": URLs are %s`
  с двумя вариантами продолжения. В core — только текст: `Check\DispatchLine::describe(string $dispatch): string` (или константы на
  `DispatcherFactory`); 3 вызова. Мелко, входит в §2.12.
- **ORM-check ×3** — две строки, существительные фреймворка. Оставить.

### 2.5. Диспетчеры очередей — ПОДТВЕРЖДЕНО: каркас в core; джобы — оставить

`MessengerDispatcher`, laravel и yii2 `QueueDispatcher`: один и тот же цикл — `array_chunk($urls, max(1, batchMaxUrls))`, `newId()`, `try {
<enqueue фреймворка> ; debug-лог } catch (Throwable) { error-лог «they are lost» }`. Различается только строка внутри `try` (стампы Messenger,
`onConnection/onQueue/delay`, `ttr/delay/priority/push`). Неоднородность: бандл логирует `array_slice($chunk, 0, $this->logUrls)`, остальные —
`$config->logSample($chunk)`; текст `dispatched to messenger as message` против `queued as job`. Дизайн: `Dispatch\BatchingDispatcher`
(§4.5) — замыкание `(list<string> $urls, string $jobId): void`, размер батча, логгер, `Config::logSample()`, существительное для лога;
`newJobId()` там же (снимает три `bin2hex(random_bytes(6))`). Адаптеры оставляют свои классы как тонкие фабрики замыкания (имена
`Queue\QueueDispatcher`, `Messenger\MessengerDispatcher` не меняются). Строк ±0, зато один текст, один генератор id, одно разбиение.
Джобы (20–32 %): Messenger бросает `RecoverableMessageHandlingException`, Laravel `release()`/`fail()`, yii2 `push()` нового — общая часть
уже в `Retry\WorkerOutcome`. Оставить.

### 2.6. `KeyFileController` бандл ~ Laravel 84 % — ОТВЕРГНУТО с причиной

Оба: `$request->getHost()` → `KeyFileResponder::bodyForKey()` → 404 или `Response($body, 200, KeyFileResponder::headers())`, 31–32 строки.
Заменить на PSR-15 `Key\KeyFileRequestHandler` можно только через мост: бандлу — `symfony/psr-http-message-bridge` плюс PSR-17 реализация
(`nyholm/psr7` у бандла только в `require-dev`), Laravel — то же самое (`Illuminate\Http\Request` наследует HttpFoundation, PSR-7 нет; `nyholm/psr7`
только в `suggest`). Две зависимости в `require` ради 31 строки — нет. Мелочь: оба контроллера берут `(responder, maxAge, varyHost)` и зовут
статический `KeyFileResponder::headers()`, yii2 — `Config::keyFileHeaders()`; можно унифицировать на `Config::keyFileHeaders()`, но это смена
конструктора публичного класса бандла (`Controller\KeyFileController` в `bc.md`) ради нуля строк — не делать.

### 2.7. `ConfigFactory` адаптера ×4 — задуманная форма; знание о пакетах ×3

Каждая — одна декларация `new Adapter\ConfigFactory(ownedOptions:, dispatchModes:, autoDispatch:, needBaseUrl:, defaults:, validate:, checkCommand:,
ignoreBlocks:)`; логика `critical + disabled`, слияние блоков, `dispatch: auto` — в core (спека 16 §2.2 выполнена). Копия — в laravel/yii2/yii3
восемь одинаковых строк: `OptionalPackage::x($flag)->installed()` ×3, `...$sitemap ? SitemapServices::options() : []` ×3,
`ignoreBlocks: [...$sitemap ? [] : ['sitemap'], …]` ×3. Четвёртый пакет = три правки в трёх адаптерах. Дизайн (§4.7, необязательный):
`OptionalPackage` получает необязательное имя статического метода опций пакета (строка, класс пакета не грузится, пока `installed()` ложно),
`OptionalPackage::ownedOptions(list<OptionalPackage>)` и `::ignoredBlocks(list<OptionalPackage>)`. −15 строк, +1 источник знания.
Бандл не трогать (его `ownedOptions` — ключи обработанного дерева).

### 2.8. `*RouteUrlResolver` ×4 — ПОДТВЕРЖДЕНО: хелпер в core, не базовый класс

В `core/src/Url/` нет ничего общего для реализаций `RouteUrlResolverInterface` (интерфейс — тир Implement, «may grow»). Четыре реализации
переизобретают: `locales(array|string): list<?string>` (bundle, yii2, yii3 — дословно 10 строк; Laravel — плюс предупреждение «locales: 'all' but
router.locales is empty», один раз на процесс), `rebase(string $url, string $root)` (laravel = yii2 дословно, 12 строк), корень для пинованного
хоста (`baseUrlFor($host) ?? 'https://' . $host` ×4), исключение `Cannot generate route "%s": %s` ×4, `No request to take the host from …` ×2.
Дизайн — статический хелпер `Url\RouteOrigin` (§4.6), тир Call: адаптеры остаются `final` и независимыми, наследования нет. Побочная выгода:
предупреждение Laravel об пустом списке локалей получают все четыре (сейчас Symfony/Yii2/Yii3 молча сворачивают правило в одну URL —
`check` предупреждает, рантайм нет). ~ −70 строк, четыре текста → один.

### 2.9. Мосты к PSR — проверены по стандарту и Packagist

| Мост | Готовое в экосистеме (Packagist, 2026-09-07) | По букве стандарта | Вердикт |
|---|---|---|---|
| yii2 `Cache\Psr16Cache` (128) + `InvalidKey` | официального нет (`yiisoft/cache` — Yii3); `yii2-extended/yii2-psr16-simple-cache-bridge` 9.0.11 от 2026-08-27, 49.6k загрузок, один мейнтейнер, GitLab; `bestyii/yii2-psr16cache` 2.8k; `aivchen/yii2-simple-cache-adapter` 1.4k | ключи: разрешены `A-Za-z0-9_.`, зарезервированы `{}()/\@:` (MUST NOT support) — регулярка `[^{}()\/\\@:]{1,64}` шире разрешённого, но запрещённое отвергает (допустимо: «implementations MAY support additional characters»); `InvalidKey implements Psr\SimpleCache\InvalidArgumentException` ✓; TTL `null|int|DateInterval` ✓ (`DateInterval` → секунды через `new DateTimeImmutable()` — часов у PSR-16 нет); `getMultiple/setMultiple/deleteMultiple` ✓; `get()` → `$default` при промахе ✓ (конверт `['v' => …]` отличает сохранённый `false`) | **оставить** (зависимость от одного мейнтейнера хуже 128 строк) |
| yii2 `Debounce\YiiCacheDebounceStore` (53) | — | дублирует `Debounce\Psr16DebounceStore` (55) над Yii-API | **удалить**: `new Psr16DebounceStore(new Psr16Cache($cache), $prefix)` в `Wiring:90`; класс не назван в `bc.md` yii2 |
| yii2 `Log\YiiLogger` (65) | официальный `yiisoft/yii2-psr-log-source` 1.0.0 (2022-10-04, 22.5k, `psr/log ^3` только — семейство держит `^2 \|\| ^3`); не интерполирует, игнорирует `exception`, одна категория; `samdark/yii2-psr-log-target` (2.0M) — обратное направление (Yii → PSR) | интерполяция `{placeholder}` — «implementors MAY replace» ✓; `exception` в контексте ✓; **нарушение**: `LEVELS[$level] ?? Logger::LEVEL_INFO` — «Calling this method with a level not defined by this specification MUST throw a `Psr\Log\InvalidArgumentException`» | **оставить, исправить уровень** (2 строки) |
| yii3 `Log\CategoryLogger` (32) | — (yiisoft/log ждёт `category` в контексте) | делегирует всё ✓ | оставить |
| laravel `Event\EventDispatcherBridge` (25) | `Illuminate\Events\Dispatcher` не PSR-14, официального моста нет | `dispatch()` возвращает событие ✓; `StoppableEventInterface` не проверяет — мост получает только `Result` (не stoppable); документировать | оставить |
| yii2 `Event\ResultDispatcher` (27) + `ResultEvent` (20) | — | возвращает событие ✓; чужие события — без слушателей (это адаптер к `Component::trigger`) | оставить |
| yii3 `Event\ObservedDispatcher` (64) | — | `isPropagationStopped()` перед вызовом observer ✓, событие возвращается ✓ | оставить |
| bundle `EventListener\FlushListener(ContainerInterface $locator)` | — | PSR-11: «Users SHOULD NOT pass a container into an object so that the object can retrieve its own dependencies» | **заменить на `service_closure('indexnowkit')`** → `Closure(): IndexNowKit` — та же ленивость, без PSR-11; конструктор публичного класса (`bc.md` бандла) — «Changed» в невыпущенном 0.15.0 |
| yii3 `Check\CacheProbe(ContainerInterface)` | — | id приходит из конфигурации — законный `get($id)` | оставить |

### 2.10. Yii2 контроллер (598 строк) — оставить; три мелочи

`ControllerOutput` (27) наследует `Symfony\...\Output\Output` и пишет через `Controller::stdout()` — `StreamOutput` обошёл бы `isColorEnabled()`
и подмену `stdout()` в тестах; оставить. `HistoryAction`/`SitemapAction` — фабрики раннеров, не копии тел (27–33 % с классами команд).
Мелочи: `SitemapAction:32` повторяет гейт `sitemap.enabled` раннера (убрать, `SitemapServices::runner()` уже передаёт `$config->enabled`);
`IndexNowController:319`, `:355` — `50` и `32` вместо констант; `HistoryAction:60` — третья копия описания debounce-стора (§1.2).

### 2.11. Пакеты `verify`, `sitemap`, `history`, `testing` — чисто, одно замечание

Своих `Check\*` с текстами «not installed» нет; `debounce.store` разбирает только `History\Adapter\HistoryServices:102` (входит в §2.12).
`testing/src/ReadmeAssertions::FAMILY_COMMANDS` — ручной список 20 имён (с Yii2-написаниями `indexnow/…`), дублирует `Definitions`, но
`testing` зависит только от core и не может читать console; оставить, записать в docblock, откуда список.

### 2.12. Тексты и константы — ПОДТВЕРЖДЕНО (таблица §1.2)

- `DebounceStoreFactory::isShared(?string $store): bool` в core (`store !== null && !in_array(memory, none)`) — 9 мест → 1, бандл перестаёт
  держать литералы `'memory'`/`'none'`.
- Laravel `about`: `OptionalPackage::notInstalledMessage()` вместо двух собранных руками строк.
- Описание debounce-стора для `status` ×3 → `History\Adapter\HistoryServices::describeStore(?string $store, string $default, Closure(string): ?object $lookup): string`
  (`memory`/`none` как есть, иначе `<id> (<ShortClass>)` или `<id> (missing)`); Laravel/Yii2/Yii3 передают свой lookup.
- Числа → константы (Laravel уходят с §2.1; Yii2 — две строки).
- Строка `dispatch "%s": URLs are %s` ×3 → §2.4.
- `Cannot generate route`, `No request to take the host from`, `baseUrlFor ?? https://` → §2.8; `newId()` → §2.5; тексты загрузчиков → §2.3.

## 3. Ревизия PSR (по тексту стандарта на php-fig.org, 2026-09-07)

Формат строки: соблюдено / нарушено / применимо-не-применено.

- **PSR-1, PSR-12 / PER-CS 2.0.** `php/.php-cs-fixer.dist.php` — `@PER-CS2.0`, `@PHP82Migration`, `declare_strict_types`, `ordered_imports`,
  `native_function_invocation` — надстройки не противоречат PER-CS. **Нарушено (только тесты)**: PSR-1 §3 — «each class is in a file by itself»;
  в `src/` — ноль файлов с двумя объявлениями, в `tests/` — **35** (core 29 — до 21 класса в `AttributeUrlResolverTest`, laravel 3, yii2 2, yii3 1),
  фикстуры внутри тестов; `autoload-dev` PSR-4 их не находит, они живут, пока загружен файл теста. Вынос в `tests/Support` — ~100 классов,
  выгода — переиспользование фикстур; PSR-1 §2.3 (side effects) файлы тестов не нарушают. `tests/Taint/entrypoints.php` — скрипт (только
  вызовы) — §2.3 соблюдён. Решение: **не сейчас**, записать как известное отклонение в тестах (§7).
- **PSR-3.** Соблюдено в core (везде `LoggerInterface`, `error_log` нет). Нарушено: yii2 `YiiLogger` — неизвестный уровень не бросает (§2.9).
  `Testing\ArrayLogger` — тест-дабл, уровень хранит как строку, не бросает — допустимо для дабла, отметить в docblock. yii3 `ObserverProvider:31`
  `trigger_error(E_USER_WARNING)` из статического контекста — логгера там нет по конструкции; дешёвая альтернатива — `ObserverProvider::set()`
  принимает и логгер, `get()` пишет `warning` в него, `trigger_error` остаётся запасным путём без логгера (5 строк).
- **PSR-4.** Соблюдено: все 11 `composer.json` — один префикс на `src/`, `Tests\` на `tests/`; `testing` — `IndexNowKit\Testing\Conformance\`.
- **PSR-6.** Применимо-не-применено, и правильно: core берёт только PSR-16; голый PSR-6 (Slim/Mezzio с `symfony/cache`) оборачивается
  `Symfony\Component\Cache\Psr16Cache`, Laminas — `laminas-cache` `SimpleCacheDecorator`. `Psr6DebounceStore` в core — нет спроса, +50 строк,
  +`psr/cache` в `require`. Одна строка в `adapters.md` §13 «PSR-6 pool → оберните в PSR-16 view вашего кеш-компонента». Бандл `CacheProbe`
  берёт `CacheItemPoolInterface` только для подписи — ок.
- **PSR-7 / PSR-17.** Соблюдено: `Psr18Transport::post()` — `withHeader()` (регистр не важен по PSR-7), заголовки ответа приводятся к нижнему регистру,
  тело читается кусками с лимитом и проверкой `Content-Length`; `KeyFileRequestHandler` — хост из `getUri()->getHost()` (PSR-7: URI несёт Host),
  404 без тела (RFC 9110 допускает), заголовки из `Config::keyFileHeaders()`. **Применимо-не-применено**: `Psr18Transport::discover()` принимает
  только клиент, фабрики всегда через `php-http/discovery` — приложение на Slim/Mezzio со своей `Psr17Factory` не может отдать её без
  `Psr18Transport::__construct()` целиком. Дизайн: необязательные `?RequestFactoryInterface $requestFactory`, `?StreamFactoryInterface $streamFactory`
  в `discover()` и `TransportFactory::lazy()` (аддитивно, §4.8).
- **PSR-11.** `ArrayResolverLocator(locate: Closure)` — четыре замыкания над контейнерами. Прямой `ContainerInterface` подошёл бы бандлу
  (ServiceLocator), Laravel (`Illuminate\Contracts\Container\Container extends Psr\Container\ContainerInterface`) и Yii3, но **не Yii2**
  (`yii\di\Container` не PSR-11, `Yii::$app` тоже) — замыкание остаётся (итог спеки 16 §4.1 подтверждён). Стандарт сам предостерегает от
  передачи контейнера в объект ради его зависимостей — что и делает бандл `FlushListener` (§2.9, исправить). Yii3 `CacheProbe` —
  законный динамический id.
- **PSR-13.** Не применимо: семейство не строит гипермедиа-ответов.
- **PSR-14.** Core — только эмиттер (`Result` через `EventDispatcherInterface`), провайдера слушателей не имеет и не должен. Мосты — §2.9.
- **PSR-15.** После волны L — core `KeyFileRequestHandler` (handler + middleware + `respond()`), Yii3 делегирует. Матрица «стек → рецепт»:
  Yii3 — handler (сделано); Slim/Mezzio/Laminas — route или middleware (описано в `adapters.md` §11); Symfony — HttpFoundation-контроллер
  (мост не окупается, §2.6); Laravel — Illuminate-контроллер (то же); Битрикс — `KeyFileResponder::bodyForPath()` над `$_SERVER` (его волна);
  plain PHP — `bin/indexnow` фазы B. Записать матрицу в `adapters.md` §11 одним абзацем.
- **PSR-16.** Соблюдено в ключах: `Psr16DebounceStore` `{prefix}sha1(url)`, `ForbiddenCounter` `<prefix>403.<host>` (IPv6 без `[]:`, аудит 0.13 S10),
  `Psr16SubmissionStore` `<prefix>history.<n>`, `RobotsCache` `<prefix>robots.<scheme>_<host>_<port>`; TTL — `int` секунд; `getMultiple()` с
  `$default = false` и `iterator_to_array`. **Нарушено**: ключ пробы `indexnowkit:check` (Laravel, Yii2) содержит зарезервированный `:`
  (§2.4). yii2 `Psr16Cache` — полнота ✓ (§2.9).
- **PSR-18.** Соблюдено: 4xx/5xx — не исключения (клиент возвращает ответ, `Response` несёт код); `ClientExceptionInterface` → `TransportException`.
  Не различаются `NetworkExceptionInterface` (retryable) и `RequestExceptionInterface` (malformed — «if and only if» запрос собран неверно):
  запросы строит core, случай практически недостижим; записать одной строкой в docblock `sendRequest()`. Редиректы: стандарт **о них молчит**
  (проверено); транспорт отключает их у клиентов, которые создаёт сам (`max_redirects: 0`, `allow_redirects: false`), а «any other discovered
  client keeps its own defaults» — конформанс H02 «ключ-файл без редиректа» на таком клиенте не проверяется честно. Записать в `adapters.md` §12.
- **PSR-20.** Соблюдено в графе (20 узлов, аудит A1/A5), `history` (`since()`, `purge` — часы), `verify` (`VerifyingSubmitter` — часы).
  **Нарушено**: sitemap `SitemapRunner::changedSince()` — `new DateTimeImmutable('-' . $option)` от стенных часов (статический метод без часов;
  `FrozenClock` невозможен); `Psr18Transport::retryAfter()` — `Response::parseRetryAfter()` без `$now` → `time()` для HTTP-date `Retry-After`
  (транспорт часов не имеет). Дизайн §4.9: `?ClockInterface` в `SitemapRunner` (sitemap 0.8.0, аддитивно) и `changedSince($option, DateTimeImmutable $now)`;
  часы в `Psr18Transport` — необязательный аргумент, низкая ценность (HTTP-date в `Retry-After` редок) — `[решение]` §6.6 «нет».
  yii2 `Psr16Cache::seconds(DateInterval)` — стенные часы допустимы (PSR-16 часов не знает).
- **Не-PSR стандарты.** `Console\ExitCode` (0/1/2) ≡ `Command::SUCCESS|FAILURE|INVALID` ✓. Composer `conflict`/`suggest` — каскад `<0.4/<0.8/<0.4` во
  всех четырёх адаптерах ✓. Keep a Changelog — все `CHANGELOG.md` с Unreleased ✓. SemVer до 1.0 — `bc.md` адаптеров описывают поверхность
  (bindings, команды, конфиг), не классы; после §2.1 Laravel `bc.md` строка «Artisan commands» получает «классы — пакетов `console`/`sitemap`/`history`»,
  как в спеке 18 §3.3 для бандла и Yii3; бандл — `FlushListener` конструктор (§2.9).

## 4. Дизайн

Все версии волны ещё Unreleased: аддитивные изменения входят в те же номера (core 0.13.0, console 0.5.0, sitemap 0.8.0, history 0.4.0);
адаптеры — в свои (symfony-bundle 0.15.0, laravel 0.15.0, yii2 0.14.0, yii3 0.1.0). Тиры — по `core/docs/bc.md`: Call (звать), Implement
(реализовывать; растёт только мажором), «may grow».

### 4.1. Laravel на классах команд (§2.1) — laravel 0.15.0

Удалить `laravel/src/Console/{Check,Config,Explain,History,KeyGenerate,Sitemap,Status,Submit,SubmitModel}Command.php` и три
`*NotInstalledCommand.php`. Оставить `Console/ModelLoader.php`. Добавить `Console/ConfigSource.php`:

```php
final class ConfigSource implements ConfigSourceInterface
{
    /** @param Closure(): array<string, array<string, mixed>> $packages the PACKAGE_BLOCKS binding */
    public function __construct(private readonly Repository $config, private readonly Application $app, private readonly OptionalPackage $sitemap, private readonly OptionalPackage $verify, private readonly OptionalPackage $history, private readonly Closure $packages) {}
    public function raw(): array;      // config('indexnow') или []
    public function build(): Config;   // ConfigFactory::build($this->raw(), (string) $this->app->environment(), $sitemap->installed(), …)
    public function packages(): array; // ($this->packages)()
}
```

Провайдер (`registerConsole()`, `registerDiagnostics()`, `boot()`):

- биндинги: `ConfigSourceInterface` → `ConfigSource`; `Console\Command\KeyGenerateCommand` → `new KeyGenerateCommand($app->make(KeyGenerateRunner::class),
  '.env', $app instanceof Foundation\Application ? $app->environmentFilePath() : $app->basePath('.env'))`; `CheckCommand` — автовайринг
  (`CheckRunner`, `ConfigSourceInterface`, `SampleOptions`); `ExplainCommand` → `new ExplainCommand($runner, $words, classArgument: 'model')`;
  `SubmitSubjectsCommand` → `new SubmitSubjectsCommand($runner, $words, classArgument: 'model')`; `Sitemap\Console\SitemapCommand` →
  `SitemapServices::command($app->make(SitemapRunner::class), 'indexnow.sitemap.url')` (раннер — `SitemapServices::runner(...)`, `enabled` из
  `SitemapConfig`); `History\Console\{History,Status}Command` → над `HistoryServices::historyRunnerFor()/statusRunnerFor()`; стабы —
  `Console\Command\{Sitemap,History,Status}NotInstalledCommand` с `notInstalledMessage()` (три существующих синглтона меняют класс).
- `SampleOptions::$sampler = $app->make(SubjectSampler::class)(...)` в биндинге `SampleOptions` (как в Yii3 `di.php`).
- `boot()`: `$this->commands([KeyGenerateCommand::class, CheckCommand::class, ConfigCommand::class, SubmitCommand::class, ExplainCommand::class,
  ...sitemap ? [SitemapCommand::class] : [SitemapNotInstalledCommand::class], ...history ? [HistoryCommand::class, StatusCommand::class] :
  [HistoryNotInstalledCommand::class, StatusNotInstalledCommand::class]])` — все с `#[AsCommand]`, ленивая карта; плюс
  `Artisan::starting(static fn (Artisan $artisan) => $artisan->addCommand(new LazyCommand('indexnow:submit-model', [],
  Definitions::submitSubjects($words, 'model')->description, false, static fn (): SubmitSubjectsCommand => $app->make(SubmitSubjectsCommand::class))))`.
- `Laravel\Sitemap\SitemapServices::commands()` / `History\HistoryServices::commands()` — возвращают классы пакетов; `StatusCommand` Laravel
  описывал debounce-стор — переезжает в `HistoryServices::statusRunner()` с `describeStore()` (§4.10).
- `docs/bc.md` Laravel: «Artisan commands» — имена/аргументы/опции те же, классы — `IndexNowKit\Console\Command\*`, `Sitemap\Console\SitemapCommand`,
  `History\Console\{History,Status}Command`; аргумент `model` сохранён. `docs/extending.md`: как заменить одну команду (`$this->app->extend(CheckCommand::class, …)`
  или своя команда с тем же именем, зарегистрированная позже). CHANGELOG «Changed» с миграцией для тех, кто наследовал `Laravel\Console\*` (классы были
  `final` — наследовать было нельзя; упоминание для тех, кто резолвил их из контейнера).
- `composer.json` Laravel: `symfony/console` не добавлять — приходит через `illuminate/console` и `indexnowkit/console`.

### 4.2. `indexnowkit/console` 0.5.0 — аддитивно, одно удаление

- `Command\SubmitSubjectsCommand::__construct(SubmitSubjectsRunner $runner, Vocabulary $words, string $classArgument = 'class')`,
  `Command\ExplainCommand::__construct(ExplainRunner $runner, Vocabulary $words, string $classArgument = 'class')` — третий именованный аргумент
  уходит в `Definitions::submitSubjects($words, $classArgument)` / `explain(...)`; `execute()` читает `$input->getArgument($this->classArgument)`.
- `Console\SubjectSampler` (§2.2), тир Call:

```php
final class SubjectSampler
{
    public const PER_CLASS = 3;
    public function __construct(private readonly SubjectLoaderInterface $subjects, private readonly IndexNowKit $indexNow) {}
    /** @return list<string> */
    public function __invoke(string $class, ?string $id): array; // тело EntitySampler как есть
}
```

- `CommandDefinition::laravelSignature()` — удалить (`[решение]` §6.2; альтернатива — `@deprecated` на один минор). `docs/bc.md` console:
  строка «Commands» дополняется `classArgument`; CHANGELOG 0.5.0 «Changed» для `laravelSignature()`.

### 4.3. `Console\AbstractSubjectLoader` (§2.3) — console 0.5.0, тир Implement, `[решение]` §6.3

```php
abstract class AbstractSubjectLoader implements SubjectLoaderInterface
{
    private readonly ClassNameResolver $classes;
    /** @param list<string> $namespaces @param class-string $marker the class or interface a subject class must extend or implement @param string $noun 'an Eloquent model' */
    public function __construct(array $namespaces, string $marker, string $noun);
    final public function resolveClass(string $class): string;             // ClassNameResolver + guard()
    final public function byIds(string $class, array $ids, Event $event): array;   // цикл found/missing над findOne()
    final public function all(string $class, int $limit, Event $event): iterable;  // max(1, $limit) → findMany()
    abstract protected function findOne(string $class, string $id, Event $event): ?object;
    abstract protected function findMany(string $class, int $limit, Event $event): iterable;
    protected function guard(string $class): string;                     // текст «"%s" is not %s (it does not extend or implement %s)…» один раз
}
```

Бандл: `EntityLoader extends AbstractSubjectLoader` — маркер не класс, а «управляемая Doctrine сущность» → предикат вместо маркера:
конструктор принимает `class-string|Closure(string): bool $accepts`. Yii3 `findMany()` — тот же `batch()`-генератор. Laravel — `withTrashed()`
внутри `findOne/findMany` по `$event === Event::Deleted`.

### 4.4. `Check\LocalesCheck` (§2.4) — core 0.13.0, тир Call

```php
final class LocalesCheck implements CheckInterface
{
    public const CODE = 'router.locales';
    /**
     * @param list<string>                 $locales   the configured list
     * @param Closure(): list<class-string> $classes   whose rules to read (Doctrine metadata, --sample-class, active_record.models)
     * @param string                       $option    the option name in the texts ('router.locales', 'framework.enabled_locales')
     * @param string|null                  $parameter the route parameter named in the ok line; null = no ok line when the list is filled
     */
    public function __construct(array $locales, AttributeReaderInterface $rules, Closure $classes, string $option = 'router.locales', ?string $parameter = null) {}
}
```

Тексты: ok — `%option%: a, b — a rule with locales: 'all' generates one URL per locale (route parameter "%p")`; warning — `%option% is empty, but
%classes% has a rule with locales: 'all': one URL in the current locale is generated instead of one per locale. List the locales in %option%.`
(с «and N more» после трёх имён — из бандла); при пустом списке и без просящих классов — ничего (как сейчас у всех трёх). Исключение из
`rules()` — пропуск класса. Адаптеры: бандл — `new LocalesCheck($enabledLocales, $reader, fn () => <классы из метаданных>, 'framework.enabled_locales')`;
Laravel — `fn () => классы из SampleOptions::$classes`, `'router.locales'`, `$localeParameter`; Yii2 — `fn () => $models`. Laravel `Check\RouterCheck` и
Yii2 `Check\RouterCheck`, бандл `Check\LocalesCheck` удаляются (или Laravel оставляет `final class RouterCheck` как 8-строчную обёртку — §6.4).
`core/docs/check-codes.md` — одна строка `router.locales` для трёх адаптеров с одним текстом.

### 4.5. `Dispatch\BatchingDispatcher` (§2.5) — core 0.13.0, тир Call

```php
final class BatchingDispatcher implements DispatcherInterface
{
    /**
     * @param Closure(list<string> $urls, string $jobId): void $enqueue the framework's push; throws on failure
     * @param string $noun what the log calls one unit: 'job', 'message'
     */
    public function __construct(Closure $enqueue, Config $config, LoggerInterface $logger = new NullLogger(), string $noun = 'job') {}
    public static function newJobId(): string;   // bin2hex(random_bytes(6))
    public function dispatch(array $urls): void; // chunk by $config->batchMaxUrls; debug «{count} URL(s) queued as {noun} {id}»; error «cannot queue … they are lost: {error}» с $config->logSample()
}
```

Адаптеры: `MessengerDispatcher`, laravel/yii2 `QueueDispatcher` становятся `final class … implements DispatcherInterface` с полем
`BatchingDispatcher` и `dispatch()` → делегат (имена классов и конструкторы сохраняются; бандл получает `Config` вместо `$logUrls`/`$batchMaxUrls` —
конструктор внутреннего сервиса `indexnowkit.dispatcher.messenger`, не в `bc.md`). Джобы зовут `BatchingDispatcher::newJobId()`; свои `newId()`
остаются как алиасы на один минор (Laravel `Queue\SubmitUrlsJob` — публичный класс).

### 4.6. `Url\RouteOrigin` (§2.8) — core 0.13.0, тир Call, статический

```php
final class RouteOrigin
{
    /** locales(): 'all' → $configured or [null] with one warning per process when the list is empty */
    public static function expand(array|string $locales, array $configured, ?LoggerInterface $logger = null, string $option = 'router.locales', ?bool &$warned = null): array;
    /** the base URL a pinned host generates on: hosts.<host>.base_url or https://<host> */
    public static function pinnedRoot(Config $config, string $host): string;
    /** scheme://host[:port] of $root over path/query/fragment of $url */
    public static function rebase(string $url, string $root): string;
    public static function generationFailed(string $route, Throwable $cause): ConfigurationException;
    public static function noRequestHost(string $where = 'a console command'): ConfigurationException;
}
```

Четыре резолвера: `locales()` → `RouteOrigin::expand(...)` (у Laravel — с логгером, как сейчас; остальные три получают необязательный
`?LoggerInterface` в конец конструктора — аддитивно); `rebase()` и `contextFor/rootFor/origin` через `pinnedRoot()`; `catch` → `generationFailed()`.

### 4.7. `Adapter\OptionalPackage` — опции пакета (§2.7) — core 0.13.0, необязательный пункт

`OptionalPackage::__construct(..., ?string $optionsProvider = null)` — строка `'IndexNowKit\Sitemap\Adapter\SitemapServices::options'`
(вызывается только при `installed()`); `sitemap()/verify()/history()` заполняют её; статические `ownedOptions(array $packages): list<string>` и
`ignoredBlocks(array $packages): list<string>`. Laravel/Yii2/Yii3 `ConfigFactory::factory()` сокращаются до одной строки на каждое.

### 4.8. `Http\Psr18Transport::discover()` и `TransportFactory::lazy()` с фабриками PSR-17 (§3, PSR-17) — core 0.13.0

`discover(?ClientInterface $client = null, ?float $timeout = null, array $extraHeaders = [], ?int $getBodyLimit = null,
?RequestFactoryInterface $requestFactory = null, ?StreamFactoryInterface $streamFactory = null)`; `TransportFactory::lazy(Config, ?Closure, array, ?int,
?RequestFactoryInterface, ?StreamFactoryInterface)`. Yii3 передаёт фабрики контейнера (`Wiring`), бандл/Laravel — ничего (discovery, как сейчас).
`adapters.md` §12: абзац про редиректы (§3, PSR-18).

### 4.9. Часы (§3, PSR-20) — sitemap 0.8.0

`SitemapRunner::__construct(..., bool $enabled = true, ?ClockInterface $clock = null)`; `changedSince(?string $option, ?DateTimeImmutable $now = null)`
— относительный интервал считается от `$now` (`$now->modify('-' . $option)`), абсолютная дата — как есть; `SitemapServices::runner()` принимает
`?ClockInterface` и передаёт часы графа. Тест на `FrozenClock` в `sitemap/tests/Unit/Console`.

### 4.10. Мелочи одной строкой

- `Debounce\DebounceStoreFactory::isShared(?string $store): bool` (core) — 9 мест.
- `Check\DebounceStoreCheck::PROBE_KEY = 'indexnowkit_check'` (core); четыре пробы пишут `set(PROBE_KEY, 1, 5)`; docblock — «writes a test key».
- `History\Adapter\HistoryServices::describeStore(?string $store, string $default, Closure $lookup): string` (history 0.4.0) — 3 места.
- `Check\DispatchLine::describe(string $dispatch): string` (core) — 3 места (текст «sent synchronously when the unit of work ends; 429/5xx are not retried» /
  «collected but never sent (drain the collector yourself)»).
- yii2 `YiiLogger::log()` — `throw new \Psr\Log\InvalidArgumentException(sprintf('Unknown log level "%s".', …))` на неизвестный уровень.
- yii2 `Wiring:90` — `Psr16DebounceStore` над `Psr16Cache`; `Debounce\YiiCacheDebounceStore` удалить; CHANGELOG «Removed».
- бандл `FlushListener(CollectorInterface, Closure $indexNow)` + `service_closure('indexnowkit')` в загрузчике.
- yii3 `ObserverProvider::set(IndexNowObserver $observer, ?LoggerInterface $logger = null)`; `get()` — `warning` в логгер, `trigger_error` только без него.
- Laravel `about` — `notInstalledMessage()`; yii2 `IndexNowController:319/:355` — константы; yii2 `SitemapAction` — гейт убрать.
- `adapters.md` §11 — матрица PSR-15; §12 — редиректы PSR-18; §13 — PSR-6 одной строкой; `Testing\ArrayLogger` docblock — «уровни не проверяет: тест-дабл».

## 5. Тесты и гейт

- Laravel: `tests/Feature/*` **без правки сценариев** (аргумент `model` сохранён, `expectsOutputToContain` работает — §2.1); новые: `ConfigSourceTest`
  (raw/build/packages), проверка, что `php artisan list` не бросает без сети и не строит транспорт (`FakeTransport` не вызывался), что
  `indexnow:submit-model` — `LazyCommand` (`$artisan->get('indexnow:submit-model')` строит по требованию). `Readme`-тест: `FAMILY_COMMANDS` не меняется.
- console: `SubjectSamplerTest` (класс без id → три URL; с id → одна; отсутствующий id → пусто), тесты `classArgument` для двух команд,
  `AbstractSubjectLoaderTest` на анонимном наследнике (found/missing, `max(1, limit)`, текст guard).
- core: `LocalesCheckTest` (три ветки + «and N more» + исключение из `rules()`), `BatchingDispatcherTest` (чанки, id в обоих логах, ошибка → «lost» с
  `logSample`), `RouteOriginTest` (`expand` с предупреждением один раз, `rebase` с портом/query/fragment, `pinnedRoot`), `DebounceStoreFactory::isShared`,
  `Psr18Transport::discover()` с явными фабриками (без `php-http/discovery` — мок `Psr18ClientDiscovery` не нужен: фабрики переданы).
- sitemap: `changedSince('1 day', FrozenClock)`; history: `describeStore` три ветки; yii2: `YiiLogger` неизвестный уровень → исключение,
  `Psr16DebounceStore` над `Psr16Cache` проходит `DebounceStoreConformance` (если есть в testing; иначе существующие тесты `YiiCacheDebounceStore`
  переезжают на новую проводку).
- Гейт полностью, как в волне L: `bin/ci` ×11 (highest, lowest — Laravel 13 lowest, dbal3), absent-сценарии sitemap/verify/history для четырёх
  адаптеров, `bin/mutation` + `bin/taint` по пяти пакетам (MSI-полы), `bin/cs`, mkdocs. Coverage-полы — перезаписать с CI, не локально.

## 6. `[решение]` — для пользователя

1. **Laravel на классах команд (§2.1, §4.1).** Да / нет. Цена ~3–4 часа, −460 строк, регрессионный риск средний (тесты покрывают все девять команд,
   но тестируемый вывод идёт через мок `OutputStyle` — проверить ручным `php artisan indexnow:check` в тестовом приложении Testbench). Рекомендация: **да**.
2. **`CommandDefinition::laravelSignature()`.** (a) удалить в 0.5.0 (console 0.4.x с ним выпущен; сторонних потребителей не видно) — «Changed»;
   (b) `@deprecated`, удалить в 0.6.0. Рекомендация: **(a)** — метод без потребителя в семействе.
3. **`AbstractSubjectLoader` (§4.3).** Да / нет / позже. −120 строк, ~2 часа, риск низкий. Рекомендация: **да, последним пунктом волны** (если время есть).
4. **`Check\LocalesCheck` в core (§4.4) и судьба `Laravel\Check\RouterCheck`.** (a) удалить класс Laravel, «Changed» в 0.15.0 (`bc.md` называл имя);
   (b) оставить 8-строчную `final class RouterCheck` над core. Рекомендация: **(a)** — 0.15.0 уже несёт «Changed» (§4.1), а обёртка ради имени —
   ещё одна копия.
5. **`BatchingDispatcher` (§4.5).** Да / нет. ±0 строк, один текст и один id; ~1.5 часа; риск — очередные тесты трёх адаптеров (есть). Рекомендация: **да**.
6. **Часы в `Psr18Transport` для `Retry-After` HTTP-date.** Рекомендация: **нет** (редкий формат, тестов на него нет, `SitemapRunner` — да).
7. **`OptionalPackage` с опциями пакетов (§4.7).** Да / нет. −15 строк, +1 место знания; ~1 час. Рекомендация: **да, если делается §6.3**; иначе отложить.
8. **Фикстуры в тестах — по классу на файл (PSR-1 §3).** Рекомендация: **нет** (35 файлов, ~100 классов, выгоды пользователю нет); записать в §7.

## 7. Не делать (рассмотрено, с цифрой)

- `CacheProbe` ×4 (30–47 %), `QueueCheck`/`DispatchCheck` (41–50 %), ORM-check ×3 (38–54 %), `SubjectReader` ×3 (43–46 %), Observer ×3 (14–33 %),
  `Wiring` ×2 (13 %), `*Services` ×2 (25–36 %), composition roots (3–12 %) — фреймворковые обёртки.
- `KeyFileController` бандл ~ Laravel 84 %: мост стоит две зависимости в `require` ради 31 строки (§2.6).
- Джобы очередей (20–32 %): три разных механизма ретрая, общее уже в `WorkerOutcome`.
- yii2 `Psr16Cache` → сторонний пакет: единственный живой кандидат — один мейнтейнер на GitLab (§2.9).
- yii2 `YiiLogger` → `yiisoft/yii2-psr-log-source`: официальный, но `psr/log ^3` только, без интерполяции и `exception` (§2.9).
- `Psr6DebounceStore`/адаптер PSR-6→PSR-16 в core: спроса нет, у всех экосистем есть свой PSR-16 view (§3).
- Yii2 контроллер как symfony-команды: `yii\console\Controller` — не `Symfony\...\Command`; `ControllerOutput` нужен ради `stdout()` (§2.10).
- `ArrayResolverLocator` на `ContainerInterface`: Yii2 не PSR-11 (§3).
- Тесты по классу на файл (PSR-1 §3): 35 файлов, известное отклонение только в `tests/`.
- Фаза B `bin/indexnow`, Битрикс — их волна.

## 8. Definition of Done волны M

- [x] §4.1 выполнен: `laravel/src/Console/` = `ModelLoader.php` + `ConfigSource.php`; `tests/Feature/*` зелёные без правки сценариев; `php artisan list`
      в Testbench не трогает транспорт; `indexnow:submit-model` — ленивая.
- [x] console 0.5.0: `SubjectSampler`, `classArgument` у двух команд, (`AbstractSubjectLoader` по §6.3), `laravelSignature()` по §6.2; четыре сэмплера адаптеров удалены.
- [x] core 0.13.0: `Check\LocalesCheck`, `Dispatch\BatchingDispatcher`, `Url\RouteOrigin`, `DebounceStoreFactory::isShared()`, `DebounceStoreCheck::PROBE_KEY`,
      `Check\DispatchLine`, `Psr18Transport::discover()`/`TransportFactory::lazy()` с фабриками, (`OptionalPackage` опции по §6.7); все девять `in_array`
      и четыре текста роутера ушли (grep пустой).
- [x] sitemap 0.8.0: часы в `SitemapRunner`; history 0.4.0: `describeStore()`; yii2 0.14.0: `YiiCacheDebounceStore` удалён, `YiiLogger` бросает; бандл 0.15.0:
      `FlushListener` на замыкании; yii3 0.1.0: `ObserverProvider` с логгером; Laravel `about` — `notInstalledMessage()`.
- [x] Документация: `bc.md` (laravel, console, bundle), `check-codes.md`, `adapters.md` §11/§12/§13, CHANGELOG каждого затронутого пакета, спека 19 —
      статус «реализовано» с отклонениями, спека 18 §9 — сноска «первый пункт опровергнут спекой 19 §2.1».
- [x] Гейт §5 зелёный; коммиты conventional без attribution; пуша нет; память `project-wave-m-leftovers` обновлена.
