# 18. Общие консольные команды семейства indexnowkit/php (волна L, после Yii3, до Битрикса)

Статус: **проект, ждёт решений §10** (2026-09-07). Основание: сравнение четырёх адаптеров после закрытия аудита 0.13 и волны K
(`docs/plans/audit-0.13.md`, спека 17 §16.5). Язык — словарь ядра (`core/docs/adapters.md` «Names»), полные слова.

## 0. Цель и границы

Yii3 — третий адаптер на `Symfony\Component\Console` (после бандла; Laravel стоит на artisan, Yii2 — на контроллере). Его
одиннадцать команд — копия команд бандла: `SubmitCommand` 96 %, `ExplainCommand` 89 %, три `*NotInstalledCommand` 84–87 %,
`HistoryCommand` 82 %, `CheckCommand` и `KeyGenerateCommand` 76 %, `SitemapCommand` 73 % (difflib по строкам, 2026-09-07).
Различаются не команды, а **как адаптер отдаёт им зависимости**. Цель волны: команды становятся классами пакетов
(`indexnowkit/console`, `sitemap`, `history`), адаптер на symfony/console только **регистрирует** их и поставляет то, что
варьируется. Следствия: минус ≈ 900 строк в бандле и yii3, четвёртый symfony/console-потребитель (Битрикс через `bin/indexnow`,
plain PHP — §8) получает команды даром, а правка команды перестаёт быть правкой в трёх местах.

Не в границах: artisan-команды Laravel (другой базовый класс, `Definitions::laravelSignature()` уже делает их тонкими),
`Yii2\Console\IndexNowController` (контроллер Yii2, `yiiOptions()`), `Check\CacheProbe` и `Check\RouterCheck` трёх адаптеров
(обёртки над кешем и роутером фреймворка, общего кода < 50 %), наблюдатели ORM.

## 1. Что есть сейчас (факты)

| | symfony-bundle | yii3 |
|---|---|---|
| базовый класс | `Symfony\Component\Console\Command\Command`, `#[AsCommand]` | тот же |
| регистрация | `$services->set(XCommand::class)->args([...])->tag('console.command')` в `IndexNowKitLoader::loadConsole()` | карта имя → класс в `config/params-console.php` (`yiisoft/yii-console`), автовайринг из `di-console.php` |
| тело | раннеры `indexnowkit/console` (`CheckRunner`, …) — сервисы `indexnowkit.console.*` | те же раннеры — определения `di-console.php` |
| аргументы и опции | `Definitions::*()->applyTo($this)` | то же |
| конфиг для `check`/`config` | сырой массив + `%kernel.environment%` в конструкторе, `ConfigFactory::build()` бандла | фасад `IndexNow`: `buildConfig()`, `options()`, `verifyConfig()`, `historyConfig()` |
| `--sample` | `SampleOptions` — сервис, сэмплер подставлен DI | `SampleOptions` из контейнера, сэмплер подставляет **сама команда** (`RecordSampler`) |
| `key:generate` | `%kernel.project_dir%/.env.local` | `getcwd()/.env` или `envFile` из определения |
| sitemap / history / status | раннеры собраны в DI пакетов (`SitemapServices::register()`, `HistoryServices::registerConsole()`) | раннеры собираются **в команде** (`SitemapServices::runner(...)`, `HistoryServices::historyRunnerFor()`, `statusRunnerFor()` с описанием debounce-стора из контейнера) |
| пакет не установлен | стаб-класс регистрируется вместо команды | стаб в карте **и** проверка `installed()` внутри команды (дважды) |
| имя команды сущностей | `indexnow:submit-entity` | `indexnow:submit-record` (Laravel: `submit-model`) |

Общее: `Definitions`, раннеры, `Vocabulary`, `ResultRenderer`, `ExitCode`, тексты `notInstalledMessage()`. Варьируется ровно
пять вещей: источник конфига, имя команды сущностей и слова, файл `.env` по умолчанию, сборка раннера пакета (sitemap/history/status),
регистрация.

## 2. Принципы

1. **Раннер — порт, команда — адаптер порта к symfony/console, фреймворковый адаптер — composition root.** Раннеры уже тир Call
   в `console/docs/bc.md`; всё, что варьируется, входит в команду **конструктором** (значения и один интерфейс), а не наследованием
   и не через контейнер. Команда не знает ни фреймворка, ни контейнера (`StatusCommand` yii3 сегодня берёт `ContainerInterface`
   ради описания стора — это уезжает в определение раннера).
2. **Ничего не собирается внутри команды.** `SitemapRunner`, `HistoryRunner`, `StatusRunner` — определения контейнера адаптера
   (как в бандле сегодня); yii3 переводит свои три сборки из команд в `di-console.php`. Проверка «пакет установлен» — только
   регистрацией: установлен → команда пакета, нет → стаб. Двойной проверки внутри команды нет.
3. **Команда пакета живёт в пакете**: `indexnow:sitemap` — в `indexnowkit/sitemap`, `indexnow:history` и `indexnow:status` —
   в `indexnowkit/history`, остальные — в `indexnowkit/console`. `console` не зависит от опциональных пакетов (как сейчас).
4. **Symfony-практики**: `final` классы, инъекция конструктором, `#[AsCommand(name:)]` там, где имя постоянное; там, где имя
   задаёт адаптер (`submit-entity`/`submit-record`, стабы) — имя в конструкторе и **ленивая регистрация тегом с атрибутом**
   `command` (`->tag('console.command', ['command' => 'indexnow:submit-entity', 'description' => …])`), чтобы
   `AddConsoleCommandPass` не инстанцировал команду при старте приложения. Описания — из `Definitions` (`setDescription()` в
   `configure()`), чтобы слово «entity»/«record» приходило из `Vocabulary`, а не из атрибута.
5. **Поверхность CLI не меняется**: имена команд, аргументы, опции, коды выхода, `--json` — те же (`Definitions` не трогаются).
   Функциональные тесты команд бандла и yii3 (`CommandsTest`, `HistoryConsoleTest`, `VerifyConsoleTest`, `*NotInstalledTest`,
   `ConsoleCommandMapTest`) должны остаться зелёными **без правок**, кроме ссылок на классы — это и есть критерий приёмки.

## 3. Дизайн

### 3.1. `indexnowkit/console` — `IndexNowKit\Console\Command\*` (тир Call) и один интерфейс (тир Implement)

```php
namespace IndexNowKit\Console;

/** What `check` and `config` read: the adapter's raw configuration, the strict build of it, the blocks of the installed optional packages. */
interface ConfigSourceInterface
{
    /** @return array<string, mixed> the raw array the adapter feeds Config::fromArray() (env placeholders resolved) */
    public function raw(): array;
    /** @throws \IndexNowKit\Exception\ConfigurationException the strict path: Adapter\ConfigFactory::build() */
    public function build(): Config;
    /** @return array<string, array<string, mixed>> block name => toArray() of the installed optional packages (`verify`, `history`, …) */
    public function packages(): array;
}
```

```php
namespace IndexNowKit\Console\Command;

#[AsCommand(name: 'indexnow:submit')]
final class SubmitCommand extends Command            { __construct(SubmitRunner $runner) }

final class SubmitSubjectsCommand extends Command    { __construct(SubmitSubjectsRunner $runner, Vocabulary $words) }
    // name = $words->submitSubjects ('indexnow:submit-entity' | 'indexnow:submit-record'), description from Definitions::submitSubjects($words)

#[AsCommand(name: 'indexnow:explain')]
final class ExplainCommand extends Command           { __construct(ExplainRunner $runner, Vocabulary $words) }

#[AsCommand(name: 'indexnow:check')]
final class CheckCommand extends Command             { __construct(CheckRunner $runner, ConfigSourceInterface $config, ?SampleOptions $samples = null) }

#[AsCommand(name: 'indexnow:config')]
final class ConfigCommand extends Command            { __construct(ConfigRunner $runner, ConfigSourceInterface $config) }

#[AsCommand(name: 'indexnow:key:generate')]
final class KeyGenerateCommand extends Command       { __construct(KeyGenerateRunner $runner, string $envFileName = '.env', ?string $envFile = null) }
    // --write-env without a value: $envFile ?? getcwd() . '/' . $envFileName; $envFileName also names the file in the help text (Definitions::keyGenerate())

final class NotInstalledCommand extends Command      { __construct(string $name, string $description, string $message) }
    // one class for the three stubs: ignoreValidationErrors(), prints $message on one line (a cron log greps it), ExitCode::FAILURE
```

Тела — те, что сегодня в бандле (они уже без фреймворка), с двумя правками: `CheckCommand` не знает про сэмплер (он в
`SampleOptions`, которую адаптер собирает целиком), `KeyGenerateCommand` — параметризованный файл.

### 3.2. Команды опциональных пакетов

```php
IndexNowKit\Sitemap\Console\SitemapCommand   { __construct(SitemapRunner $runner) }          // #[AsCommand('indexnow:sitemap')]
IndexNowKit\History\Console\HistoryCommand   { __construct(HistoryRunner $runner) }          // #[AsCommand('indexnow:history')]
IndexNowKit\History\Console\StatusCommand    { __construct(StatusRunner $runner) }           // #[AsCommand('indexnow:status')]
```

Тела — из бандла. Проверка `sitemap.enabled` (yii3 печатает `sitemap.enabled is false.` и `INVALID`) — сверить, где она у
бандла; если у раннера — оставить там, если только в yii3 — перенести в `SitemapRunner` (одно место для всех).

### 3.3. Что делает адаптер

**symfony-bundle** (`IndexNowKitLoader::loadConsole()`, `SitemapServices::register()`, `HistoryServices::registerConsole()`):
- `Command\*` бандла удаляются; в DI регистрируются классы пакетов с теми же аргументами (`indexnowkit.console.*` — раннеры,
  как сейчас). Для `SubmitSubjectsCommand` и трёх `NotInstalledCommand` — тег с `command`/`description`.
- `ConfigSourceInterface` → `DependencyInjection\ConsoleConfigSource(array $raw, string $environment, array $packages)` — сегодняшние
  три аргумента `CheckCommand`/`ConfigCommand` в одном объекте; `build()` зовёт `ConfigFactory::build()` бандла.
- `KeyGenerateCommand`: `envFileName: '.env.local'`, `envFile: '%kernel.project_dir%/.env.local'`.
- `Command\*` удалены из «Public classes» в `docs/bc.md` бандла; в `extending.md` — как заменить одну команду (decorate по классу
  пакета или своя команда с тем же именем).

**yii3** (`config/params-console.php`, `config/di-console.php`, `config/di.php`):
- `Console\*` yii3 удаляются, кроме ничего; карта команд ссылается на классы пакетов. Стабы: три id определений
  (`indexnow.command.sitemap-absent` и т. п.) с `NotInstalledCommand` и своими аргументами — `yiisoft/yii-console` резолвит
  значение карты через контейнер по строке, id не обязан быть классом (проверить на установленной версии).
- `di-console.php`: определения `SitemapRunner`, `HistoryRunner`, `StatusRunner` (через `SitemapServices::runner()`,
  `HistoryServices::historyRunnerFor()`, `statusRunnerFor()` с описанием debounce-стора — сегодняшний
  `StatusCommand::debounceDescription()` переезжает в фабрику определения), только при установленном пакете (та же ветка,
  что в карте). `SampleOptions` в `di.php` — с сэмплером (`RecordSampler`) из фабрики определения, не из команды.
- `Console\ConfigSource(IndexNow $indexNow)` — реализация интерфейса над фасадом (`options()`, `buildConfig()`, блоки
  `verify`/`history` по `*Installed()`).
- `KeyGenerateCommand`: `envFileName: '.env'`, `envFile: null` (или путь из определения, как сейчас).
- `docs/bc.md` yii3: строка «Console commands» — имена и опции те же, классы — пакетов.

**laravel, yii2** — без изменений в коде; `require` `indexnowkit/console ^0.5` (каскад версии).

### 3.4. Что варьируется и как входит

| Варьируется | Как входит в команду | bundle | yii3 |
|---|---|---|---|
| источник конфига | `ConfigSourceInterface` | `ConsoleConfigSource(raw, env, packages)` | `ConfigSource(IndexNow)` |
| слова и имя команды сущностей | `Vocabulary` (`subject`, `submitSubjects`, …) | сервис `indexnowkit.console.vocabulary` | определение `Vocabulary` |
| `--sample` | `?SampleOptions` (сэмплер уже внутри) | сервис `indexnowkit.check.samples` | определение с фабрикой |
| файл `.env` | `$envFileName`, `$envFile` | `.env.local`, `%kernel.project_dir%` | `.env`, `null` |
| раннеры пакетов | `SitemapRunner`, `HistoryRunner`, `StatusRunner` | DI пакетов | `di-console.php` |
| регистрация | вне команды | тег `console.command` | карта `params-console.php` |
| «не установлен» | `NotInstalledCommand(name, description, message)` | вместо команды пакета | вместо команды пакета |

## 4. BC и версии

- `indexnowkit/console` **0.5.0** (сейчас 0.4.2 Unreleased): новые классы `Command\*` (тир Call, конструкторы — именованные
  аргументы, растут только аддитивно), `ConfigSourceInterface` (тир Implement: методы не добавляются без мажора;
  до 1.0 — как у остальных Implement). `docs/bc.md` console — строка «Commands»: классы, их имена, что `#[AsCommand]`-имя —
  контракт, описание — нет.
- `indexnowkit/sitemap` 0.8.0 (Unreleased, аддитивно), `indexnowkit/history` **0.4.0** (сейчас 0.3.1 Unreleased; новые классы —
  минор по правилу семьи), оба требуют `console ^0.5`.
- `symfony-bundle` 0.15.0 (Unreleased): «Changed» — `Command\*` удалены, классы пакетов, миграция для тех, кто декорировал
  команды по классу (`docs/bc.md` обещал `Command\*` как публичные классы — это ломающее, потому в тот же невыпущенный минор).
- `yii3` 0.1.0 (Unreleased): без записи «Changed» — версии в мире нет; `docs/bc.md`, README (список классов, если есть).
- `laravel`, `yii2`, `doctrine`: только `console ^0.5` в `require`/`require-dev`.
- Каскад тегов не меняется: core → console, testing → sitemap, verify, history → doctrine → symfony-bundle, laravel, yii2 → yii3.

## 5. Тесты

- `console/tests/Unit/Command/*Test.php` — `Symfony\Component\Console\Tester\CommandTester` над каждой командой с раннерами на
  двойниках ядра (`FakeTransport`, `ArrayLogger`, как в `RunnersTest`): аргументы и опции доходят до раннера в нужных типах
  (`--host` список и строка, `--purge` без значения / со значением, `--write-env` без значения → `envFile`, `--sample-class`
  список), коды выхода, `--json`; `NotInstalledCommand` игнорирует любые аргументы и печатает одну строку; `SubmitSubjectsCommand`
  берёт имя из `Vocabulary`. `ConfigSourceInterface` — двойник в тесте.
- sitemap/history: по одному тесту команды на `CommandTester` над раннером с in-memory стором / `FakeTransport`.
- Бандл и yii3: существующие функциональные тесты команд остаются зелёными без правок тел; `ContainerShapeTest` бандла и
  `WiringTest`/`ConsoleCommandMapTest` yii3 — обновить имена классов; `ReadmeAiNotesTest` — без изменений (имена команд те же).
- `testing`: `ReadmeAssertions::FAMILY_COMMANDS` не меняется. Конформанс H01–H06 не затронут.
- Гейт семьи: `bin/ci` console, sitemap, history, symfony-bundle (highest + `symfony64` + `symfony8`), yii3 (highest + lowest),
  laravel, yii2, doctrine (constraint), `optional-packages-absent` локально для бандла и yii3 (стабы регистрируются без пакетов —
  именно этот путь меняется), `bin/cs`, `bin/mutation console` и `bin/taint console` (console в матрице T20; harness
  `tests/Taint/entrypoints.php` дополнить `CheckCommand`/`ConfigCommand` через `ConfigSourceInterface` с `$_POST`),
  `bin/docs-collect` + `mkdocs --strict`, `composer validate` ×затронутые.

## 6. Документация

- `console/README.md` и `docs/`: раздел «Commands» — «для приложения на symfony/console: зарегистрируй классы, дай раннеры и
  `ConfigSourceInterface`» с примером на голом `Symfony\Component\Console\Application` (это же — заготовка §8).
- `core/docs/adapters.md` §«Console»: правило «команда — в пакете, адаптер регистрирует» рядом с существующим «тексты `check` —
  в пакете». Битрикс и следующие адаптеры на symfony/console пишутся по нему.
- Бандл `extending.md`, yii3 `extending.md`: как заменить/декорировать команду; `bc.md` обоих; README-разделы для ассистентов
  (`ReadmeAiNotesTest` держит список команд).
- `php/CHANGELOG.md`, CHANGELOG'и console, sitemap, history, symfony-bundle, yii3, laravel, yii2, doctrine (constraint).
- Спека 17 §16 — абзац «волна L»; спека 12 (Symfony) и 15 (Yii) — строка о том, где живут команды.

## 7. Риски

- **Ленивые команды Symfony**: без `#[AsCommand]`-имени команда регистрируется не лениво (инстанцируется на старте `bin/console`).
  Для `SubmitSubjectsCommand` и стабов — атрибут `command` в теге обязателен; тест: `ContainerShapeTest` проверяет, что все
  теги `console.command` несут `command`, либо класс имеет `#[AsCommand]`.
- **yii-console и id определений для стабов**: три имени → один класс с разными аргументами; проверить, что
  `yiisoft/yii-console` (2.4.x) отдаёт команду по строковому id контейнера. Запасной вариант: три крошечных подкласса не
  допускаются (`final`) — тогда стаб-класс с фабрикой `NotInstalledCommand::for(OptionalPackage $package, string $name)` и три
  определения-замыкания.
- **`SampleOptions` — мутируемый объект**: команда пишет `urls`/`classes` в общий сервис; так и сегодня (в обоих адаптерах),
  поведение не меняется, но тест на «два вызова `check` в одном процессе» стоит добавить (RoadRunner в Yii3).
- **`kernel.project_dir` и `getcwd()`**: семантика `--write-env` без значения сохраняется буквально в каждом адаптере.
- Тексты описаний команд (`AsCommand description`) — не API, но `ReadmeAiNotesTest` и docs-сайт их цитируют: сверить после
  переноса, что описание бандла и yii3 совпало с `Definitions` (в yii3 сегодня «record», в бандле «entity» — и то и другое из
  `Vocabulary`).

## 8. Фаза B (отдельное решение): `bin/indexnow` для plain PHP и CMS

После §3 команды не знают фреймворка; голое приложение — тридцать строк: `Console\ConsoleApplication::fromEnv()` (или
`create(IndexNowKit $kit, ConfigSourceInterface $config, Vocabulary $words, ...)`) собирает `IndexNowKit::create(Config::fromEnv())`,
раннеры, `SampleOptions` без сэмплера, `StatusRunner`/`HistoryRunner` при установленных пакетах и отдаёт
`Symfony\Component\Console\Application` с семью-девятью командами; `vendor/bin/indexnow check`, `submit`, `key:generate`,
`config`, `sitemap`, `history`, `status` работают в любом PHP-приложении с `INDEXNOW_*` (Битрикс, WordPress-агенты, cron на
голом хостинге). `explain` и `submit-entity` — только с `SubjectLoaderInterface`, которого у голого приложения нет: не
регистрируются. Цена: `bin/indexnow` в `console/composer.json` (`bin`), `symfony/dotenv`? — нет, `Config::fromEnv()` читает
`getenv()`; документ `console/docs/standalone.md`. Рекомендация: **делать в волне Битрикса**, когда станет ясно, нужен ли
Битриксу именно этот путь (спека 14), но конструкторы §3.1 проектировать так, чтобы фаза B не потребовала их менять
(единственная точка — `ConfigSourceInterface`: реализация `EnvConfigSource(prefix)` в console).

## 9. Не делать (рассмотрено)

- **PSR-15 обработчик ключ-файла в core** (`Key\KeyFileRequestHandler` над `KeyFileResponder::bodyForPath()` и PSR-17): 56 строк
  у yii3, но Yii3 берёт ключ из аргумента маршрута (`CurrentRoute`, произвольный `key_file.pattern`), а не из пути, так что
  yii3 обработчик не заменит; ценность — только для будущего PSR-15 адаптера (Slim, Mezzio), которого нет. YAGNI; вернуться,
  когда такой адаптер появится. Зависимость `psr/http-server-handler` в core не добавлять заранее.
- **Общий `CacheProbe`/`RouterCheck`**: 26–98 строк, 39–55 % сходства — фреймворковые обёртки; общий код уже в core (`Check\*`).
- **Перевод Laravel на symfony/console-классы**: artisan-команды обязаны наследовать `Illuminate\Console\Command`; обёртки уже
  тонкие (28–63 строки), `Definitions::laravelSignature()` — их общий код.
- **`Wiring`-паттерн yii3 для провайдера Laravel** (610 строк против 435): переписывание композиции без выгоды пользователю;
  критерий формы `Services` пройден (§16.4), третий контейнерный адаптер решит, нужен ли общий каркас.

## 10. `[решение]`

1. **Когда**: (а) **до пуша текущей волны** — все затронутые версии ещё Unreleased (console 0.4.2 → 0.5.0, history 0.3.1 → 0.4.0,
   бандл 0.15.0 уже несёт «Changed», yii3 0.1.0 не выйдет с дубликатами); цена — ещё один день до пуша и повторный полный гейт;
   (б) после релиза волны — тогда bundle 0.16.0, yii3 0.2.0 с «Changed» через неделю после 0.1.0. Рекомендация — (а).
2. **Фаза B (`bin/indexnow`)**: (а) в волне Битрикса (рекомендация); (б) сразу, как часть L; (в) не делать.
3. **Имя интерфейса и место**: `Console\ConfigSourceInterface` в `indexnowkit/console` (рекомендация) или в core
   (`Adapter\ConfigSourceInterface`) — в core он был бы виден Laravel/Yii2, которым не нужен.

## 11. Definition of Done

- В бандле и yii3 нет ни одного класса `extends Symfony\Component\Console\Command\Command`; `grep -rn "extends Command" packages/{symfony-bundle,yii3}/src` пуст.
- `console/src/Command/` — семь классов + `ConfigSourceInterface`; `sitemap/src/Console/SitemapCommand.php`;
  `history/src/Console/{HistoryCommand,StatusCommand}.php`; тесты на `CommandTester` для каждой.
- Функциональные тесты команд бандла и yii3 зелёные без правки сценариев; `optional-packages-absent` для обоих зелёный локально.
- `ContainerShapeTest`: каждый `console.command` ленив (атрибут `command` или `#[AsCommand]`).
- Версии и `require` по §4; CHANGELOG'и; `bc.md` console/бандла/yii3; `adapters.md` §Console; спеки 12/15/17.
- Полный гейт семьи зелёный (§5); коммиты conventional (`feat(console)`, `feat(sitemap)`, `feat(history)`, `refactor(symfony-bundle)`,
  `refactor(yii3)`, `chore(deps)` для констрейнтов, `docs(spec)`); пуш/релиз — отдельным «действуй».
