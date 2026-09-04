# 15. PHP: `indexnowkit/yii2` и `indexnowkit/yii3`

Два пакета, один документ: Yii2 и Yii3 — разные фреймворки с разными ORM (`yii\db\ActiveRecord` и
`yiisoft/active-record`), но решения, которые отличают их от Laravel и Symfony, общие: **ни один из них не даёт
хука на commit транзакции**, и в обоих URL резолвятся в момент события ActiveRecord.

| | `indexnowkit/yii2` | `indexnowkit/yii3` |
|---|---|---|
| Namespace | `IndexNowKit\Yii2` | `IndexNowKit\Yii3` |
| Фреймворк | `yiisoft/yii2 ^2.0.45`, PHP ≥ 8.2 | `yiisoft/active-record ^1.0`, `yiisoft/db ^2.0`, `yiisoft/di ^1.2`, `yiisoft/config ^1.5`, `yiisoft/router ^3.0 \|\| ^4.0`, `yiisoft/yii-console ^2.0`, `yiisoft/yii-event ^2.0`, `yiisoft/yii-http ^1.0`; PHP ≥ 8.2 |
| Точка входа | компонент `indexnow` + `bootstrap` | конфиг-плагин `yiisoft/config` (`config/*.php` пакета) |
| Хук ORM | `IndexNowBehavior` на классе AR (или список `models`) | атрибут `#[IndexNowEvents]` + `EventsTrait` AR |
| Консоль | `php yii indexnow/<action>` (свой контроллер, вывод через `SymfonyStyle`) | `./yii indexnow:<command>` (symfony/console, как в бандле) |
| Очередь | `yiisoft/yii2-queue` (опционально) | нет стабильной (`yiisoft/queue` не выпущен): `sync`/`none`, `DispatcherInterface` заменяем |
| Дебаунс | `yii\caching\CacheInterface` (`YiiCacheDebounceStore`) | PSR-16 из контейнера (`Psr16DebounceStore`) |
| Тесты | PHPUnit 11, `yii\web\Application`/`yii\console\Application` в памяти, sqlite | PHPUnit 11, `yiisoft/di` контейнер из конфигов пакета, sqlite (`yiisoft/db-sqlite`) |

Оба: `indexnowkit/core ^0.3`, phpstan 9 + strict-rules, php-cs-fixer монорепо, split-репо `php-yii2`, `php-yii3`.

## Почему Yii, и почему вместе

Yii2 — единственный крупный PHP-фреймворк без единого IndexNow-пакета на Packagist, и его аудитория (RU/CIS)
совпадает с аудиторией Яндекса. Yii3 стабилизировался в 2025–2026 (`active-record` 1.x, `db` 2.x, `app` 1.4) и
повторяет для нового поколения тот же вакуум. Общий документ, потому что общая часть — commit-safety без хуков и
семантика событий AR — важнее различий в проводке.

## Commit-safety без хука на commit: verify-on-commit

Yii2 `yii\db\Connection` даёт `EVENT_BEGIN_TRANSACTION`, `EVENT_COMMIT_TRANSACTION`, `EVENT_ROLLBACK_TRANSACTION`
**только для внешней** транзакции; savepoint'ы вложенных `beginTransaction()` проходят через
`Schema::createSavepoint()` без событий. Yii3 `yiisoft/db` не даёт событий транзакций вообще, а `Command`,
`Transaction` и `Connection` драйверов `final` — перехват SQL `SAVEPOINT`, как в Doctrine-адаптере, невозможен.
Декорировать `ConnectionInterface` (60+ методов, растёт в минорных) — хрупко.

Решение общее для обоих и живёт в core: `Transaction\VerifyingStaging`.

1. В момент события AR адаптер резолвит URL (старое состояние ещё живо) и смотрит `getTransaction()` соединения.
   Нет активной транзакции → autocommit уже случился → URL сразу в `Collector` (как `afterCommit()` вне транзакции
   в Laravel).
2. Активная транзакция → `staging->stage($connection, $verifier, $urls)`, где `$verifier` — замыкание,
   которое **перечитывает строку по первичному ключу** и отвечает, состоялось ли изменение:
   - created / updated / renamed: строка существует **и** изменённые поля равны значениям после изменения
     (loose-сравнение приведённых к строке значений; для created сравниваются все записанные поля);
   - deleted: строки нет.
3. Доставка: `staging->flush($connection)` выполняет проверки, отдаёт в `Collector` URL прошедших, пишет `debug`
   про отброшенные (`indexnow: discarding N staged URL(s), change not committed`). Не прошедшая проверка тянет
   за собой **все** URL своего субъекта (в том числе `via`-страницы и старый URL при rename): нельзя объявить
   старую страницу удалённой, если rename откатился.
4. Когда вызывается `flush()`:
   - Yii2: на `EVENT_COMMIT_TRANSACTION` соединения (обработчик вешается лениво при первом `stage()` на это
     соединение, `WeakMap`); `EVENT_ROLLBACK_TRANSACTION` → `discard()` без запросов;
   - Yii3: событий нет — в конце запроса (`Yiisoft\Yii\Http\Event\AfterEmit`) и на завершении консольной команды
     (`Yiisoft\Yii\Console\Event\ApplicationShutdown`), плюс явный `$indexNow->flush()` для долгих команд.
     Если транзакция всё ещё активна, проверка читает через то же соединение и видит незакоммиченные данные —
     это ожидаемо: незакрытая транзакция в конце запроса — ошибка приложения, не адаптера.

Стоимость: один `SELECT ... WHERE pk = ?` на субъект, только для изменений внутри явной транзакции. Что это даёт:
A02, A05, A05b, A05c проходят без единого хука во фреймворке и без изменения конфигурации соединения. Что теряется по
сравнению с afterCommit: доставка не «в момент commit», а на commit-событии (Yii2) или в конце запроса (Yii3) — для
IndexNow разницы нет, коллектор всё равно сбрасывается в конце запроса. Документированный компромисс: update внутри
откатанной транзакции, где значения полей случайно совпали с записанными (`UPDATE ... SET title = title`), пройдёт
проверку и уйдёт как лишнее обновление существующей страницы — безвредно.

`TransactionStaging` (frames/savepoints) остаётся для Doctrine; `VerifyingStaging` — для ORM без сигналов. Оба
тестируются в core, адаптеры проверяются китом `OrmConformanceTestCase`.

## События ActiveRecord

| Сценарий | Yii2 (`yii\db\BaseActiveRecord`) | Yii3 (`yiisoft/active-record` + `EventsTrait`) |
|---|---|---|
| created | `EVENT_AFTER_INSERT` → `changes()->created()` | `AfterInsert` → `created()` |
| updated | `EVENT_AFTER_UPDATE`: `AfterSaveEvent::$changedAttributes` = **старые** значения изменённых полей, AR уже новый → `changeSet = [f => [old, new]]`; `renamed($ar, $changeSet, $previous, $selfFields)` + `updated()`; `$previous` = клон с `setAttributes($changedAttributes)`; пустой `changedAttributes` (save без изменений) → ничего | `BeforeUpdate`: снимок `oldValues()` и `newValues()` (после `AfterUpdate` `oldValues` уже перезаписаны) → в `AfterUpdate` то же, что слева; `$count = 0` (строка не изменилась, optimistic lock) → ничего |
| deleted | `EVENT_BEFORE_DELETE` резолвит URL, пока строка и отношения живы (`WeakMap` по объекту); `EVENT_AFTER_DELETE` передаёт; AR после удаления ещё хранит атрибуты (`setOldAttributes(null)`), поэтому без `beforeDelete` тоже работает | `BeforeDelete` / `AfterDelete($count)`; `$count = 0` → ничего (строки не было) |
| `link()`/`unlink()` (A20) | junction-строка пишется командой, событий владельца нет | то же |
| `updateAll()`/`deleteAll()`/`updateAttributes()`/`updateCounters()` (A13) | событий нет | событий нет |

Рецепт A20 в обоих: после `link()`/`unlink()` сохранить владельца с изменённой меткой времени
(`$post->updated_at = time(); $post->save(false)`) — `AFTER_UPDATE` с `updated_at` в changeSet доходит до правила
без фильтра `fields`; либо `Yii::$app->indexnow->submitRecord($post)`. Кит-драйверы делают именно это. A13 — ручной
путь `submitRecords()` / консольная команда `submit-record`.

`self` в `params`: Yii не имеет route model binding, поэтому `self` = значение первичного ключа (`getPrimaryKey()`
/ `primaryKeyValue()`); составной ключ → `ConfigurationException` «укажите параметры явно». `selfFields` для
rename — имена колонок первичного ключа.

Чтение accessor'ов: атрибуты AR не PHP-свойства (Yii2 `__get`, Yii3 `get()`), поэтому оба пакета регистрируют
`SubjectReaderInterface` через `ParamExtractor::registerReader()`: Yii2 `ActiveRecordSubjectReader` —
`hasAttribute()`/`getAttribute()`, отношение через `getRelation($name, false)` (loaded или lazy); Yii3 —
`hasProperty()`/`get()`, отношения `relationQuery($name)`/`relation($name)`. Незнакомое имя → к DSL core
(методы `isPublished()`), потом `ConfigurationException` (A10).

## `indexnowkit/yii2`

### Установка и конфигурация

```php
// config/web.php и config/console.php
'bootstrap' => ['indexnow'],
'components' => [
    'indexnow' => [
        'class' => \IndexNowKit\Yii2\IndexNowComponent::class,
        'options' => [                       // дерево 02 + Yii-блоки, как config/indexnow.php в Laravel
            'key' => getenv('INDEXNOW_KEY'),
            'base_url' => 'https://www.example.com',
            'dispatch' => 'auto',            // queue при наличии компонента queue, иначе sync
        ],
    ],
],
```

Компонент — `yii\base\Component implements BootstrapInterface`. `options` = `Config::OPTIONS` плюс carve-out
(`ConfigFactory::YII_OPTIONS` + `SitemapConfig::OPTIONS` как `ownedOptions` `Adapter\ConfigFactory` core; неизвестные ключи → `warning`):

| Блок | Значение |
|---|---|
| `dispatch` | `auto` (по умолчанию) → `queue`, если `Yii::$app->has(queue.component)`, иначе `sync`; явные `sync`/`queue`/`none` |
| `queue.{component,ttr,delay,priority}` | id компонента `yii2-queue` (`queue`), TTR job (300), задержка и приоритет push |
| `debounce.store` | `cache` (компонент `cache`), id другого кеш-компонента, `memory`, `none` |
| `http.client` | id компонента / класс PSR-18 клиента (`null` = `LazyTransport` + discovery) |
| `key_file.{enabled,pattern,cache_max_age}` | URL-правило `<key:[A-Za-z0-9-]{8,128}>.txt` → `indexnow/key-file/index`; требует `urlManager.enablePrettyUrl` (иначе `check` предупреждает) |
| `router.{languages,language_parameter,set_app_language}` | список для `locales: 'all'`; параметр маршрута с языком (`language`), добавляется при `$locale !== null`; на время генерации `Yii::$app->language` переключается |
| `active_record.enabled` | `false` = behavior и `models` не вешаются |
| `models` | список классов AR без `IndexNowBehavior`: class-level `Event::on(Class, EVENT_*, ...)` |
| `sitemap.*`, `logging.category` | как в бандле; лог — `Yii::getLogger()` через `yii\log\Logger` → PSR-3 адаптер пакета с категорией `indexnow` |

Невалидный конфиг: `critical` в лог + `Config(enabled: false, dryRun: true)`, точная ошибка в `php yii indexnow/check`
(как в бандле и Laravel). `dispatch: queue` без `base_url` — ошибка.

### Bootstrap

`IndexNowComponent::bootstrap($app)`:

- регистрирует `ParamExtractor::registerReader(new ActiveRecordSubjectReader())`;
- модуль `indexnow` (`IndexNowKit\Yii2\Module`): в web — контроллер `key-file`, в консоли — контроллер `indexnow`
  с действиями; если приложение уже объявило модуль с этим id — не трогаем;
- web: `urlManager->addRules([key_file.pattern => 'indexnow/key-file/index'])` при `key_file.enabled` и
  `enablePrettyUrl`; flush на `Response::EVENT_AFTER_SEND` (после отправки ответа, не внутри);
- console: flush на `Application::EVENT_AFTER_REQUEST`; при наличии `yii2-queue` — class-level
  `Event::on(Queue::class, EVENT_AFTER_EXEC|EVENT_AFTER_ERROR)` → flush после каждой job воркера;
- `models` → `Event::on(Class, BaseActiveRecord::EVENT_AFTER_INSERT|AFTER_UPDATE|BEFORE_DELETE|AFTER_DELETE, ...)`.

Компоненты core собираются лениво внутри `IndexNowComponent` (`kit()`, `submitter()`, `collector()`, …) — в Yii2
нет DI-контейнера уровня приложения для интерфейсов; замена частей — через свойства компонента
(`transport`, `debounceStore`, `dispatcher`, `urlResolver`, `checks`) с `Instance::ensure`.

### Behavior

```php
#[IndexNow(route: 'post/view', params: ['slug' => 'slug'], when: 'published')]
final class Post extends ActiveRecord
{
    public function behaviors(): array
    {
        return [IndexNowBehavior::class];
    }
}
```

`IndexNowBehavior::events()` → 4 события AR на `IndexNowObserver` компонента (singleton, WeakMap для
`beforeDelete`). Один observer на все классы; behavior — только регистрация.

### Мост роутера

`YiiRouteUrlResolver`: `Url::toRoute` не используется (глобальное состояние); напрямую
`$urlManager->createAbsoluteUrl([$route] + $params)`. `route` в атрибуте — Yii-маршрут `controller/action`
(или `module/controller/action`), `params` — GET-параметры правила. В консоли `UrlManager` без `hostInfo`/`baseUrl`
бросает `InvalidConfigException` — мост подставляет их из `base_url` перед генерацией (`setHostInfo`/`setBaseUrl`
на клоне, чтобы не менять компонент). `host:` правила → rebase готового URL на `hosts.<host>.base_url` /
`https://<host>` (как в Laravel). `$locale` → `router.language_parameter` в params + `Yii::$app->language` на время
генерации при `set_app_language`.

### Консоль

`php yii indexnow/check [--live] [--host=] [--probe-url=]`, `indexnow/submit <url>... [--force] [--dry-run] [--json]`,
`indexnow/submit-record <class> [ids...] [--event=] [--limit=] [--explain] [--force] [--dry-run] [--json]`,
`indexnow/explain <class> <id> [--event=]`, `indexnow/sitemap [sitemap] [--changed-since=] [--allow-foreign-hosts]
[--force] [--dry-run] [--json]`, `indexnow/key-generate [--length=] [--alphanumeric] [--write-env[=FILE]] [--force]`.

Тела — `Console\*Runner` core над `SymfonyStyle(new ArrayInput([]), new StreamOutput(STDOUT))`; пакет требует
`symfony/console` (Yii2 его не тянет). `ActiveRecordLoader implements SubjectLoaderInterface`: `<class>` — FQCN или
короткое имя в `app\models\`; `Vocabulary('record', 'records', 'php yii', 'indexnow/submit-record', ...)`.
`--write-env` по умолчанию пишет в `.env` рядом с `@app` (Yii2-приложения обычно на `vlucas/phpdotenv`).

### Очередь

`yiisoft/yii2-queue` в `suggest`. `SubmitUrlsJob implements RetryableJobInterface`: `execute()` → `submit()`;
ретраибельные результаты → бросает `RetryableSubmissionException`, `canRetry($attempt, $error)` = `attempt <
retry.max_attempts` и ошибка ретраибельна; задержка между попытками — драйверная (`ttr`/`attempts`
очереди), `Retry-After` учесть нельзя (у yii2-queue нет `release($delay)`) — документировано; финальные 400/403/422 →
`error` в лог, без исключения (иначе очередь ретраила бы бесполезно). `QueueDispatcher` → `Yii::$app->get(queue)->ttr()->delay()->priority()->push($job)`.

### Проверки `check`

`QueueCheck` (компонент существует; `SyncQueue` — без ретраев), `Check\DebounceStoreCheck` core с `CacheProbe` (компонент дебаунса; core 0.4, раньше `CacheCheck`), `UrlManagerCheck`
(`enablePrettyUrl` для файла ключа, правило добавлено), `ActiveRecordCheck` (behavior/`models` активны),
`Sitemap\Check\SitemapSpoolCheck` пакета `indexnowkit/sitemap`, а без него — `Check\StaticCheck` core со строкой
`sitemap: not installed (…)`.

`indexnowkit/sitemap` — `suggest` (yii2 0.4.0, спека 16 §1.5/§1.7): предикат `Sitemap\SitemapSupport::installed()`,
sitemap-части в `Sitemap\SitemapServices` и `Console\SitemapAction`; без пакета `indexnow/sitemap` (опции
по-прежнему принимаются) печатает текст установки и возвращает `ExitCode::FAILURE`, `sitemapConfig()`/`sitemapSource()`
бросают `LogicException` с тем же текстом, блок `sitemap` в опциях игнорируется. Yii3 повторяет форму.

## `indexnowkit/yii3`

### Конфиг-плагин

`composer.json` пакета: `extra.config-plugin` → `params`, `di`, `di-web`, `di-console`, `params-console`, `events-web`,
`events-console`, `routes`, `bootstrap`. Приложение на `yiisoft/app` подхватывает всё без правок; `params.php`:

```php
'indexnowkit/yii3' => [
    'key' => $_ENV['INDEXNOW_KEY'] ?? null,
    'base_url' => null,
    'dispatch' => 'sync',
    // ... дерево 02 + Yii3-блоки
    'debounce' => ['per_url' => 600, 'store' => 'cache'],     // cache = Psr\SimpleCache\CacheInterface из контейнера
    'http' => ['timeout' => 10, 'client' => null],             // id PSR-18 клиента в контейнере
    'key_file' => ['enabled' => true, 'pattern' => '/{key:[A-Za-z0-9-]{8,128}}.txt', 'cache_max_age' => 300],
    'router' => ['locales' => [], 'locale_parameter' => '_language'],
    'active_record' => ['enabled' => true],
    'sitemap' => [...],
],
```

`di.php` строит `Config` (`ConfigFactory` с `critical + disabled` при ошибке), `IndexNowKit`, все интерфейсы core как
отдельные definitions (приложение переопределяет любой в своём `di/`), `IndexNowObserver`, `VerifyingStaging`,
`SubjectLoaderInterface` (`ActiveRecordLoader`), `Console\*Runner`, `Vocabulary('record', 'records', './yii',
'indexnow:submit-record', ...)`. `di-console.php` — команды; `params-console.php` — `yiisoft/yii-console.commands`.
`routes.php` — `Route::get(key_file.pattern)->action(KeyFileHandler::class)->name('indexnow/key-file')`.
`events-web.php` — `AfterEmit` → `FlushListener`; `events-console.php` — `ApplicationShutdown` → `FlushListener`.
`bootstrap.php` — `ParamExtractor::registerReader(new ActiveRecordSubjectReader())` и
`ObserverProvider::set($container->get(IndexNowObserver::class))`.

### Хук AR

```php
#[IndexNow(route: 'post/view', params: ['slug' => 'slug'], when: 'published')]
#[IndexNowEvents]
final class Post extends ActiveRecord
{
    use EventsTrait;
}
```

`IndexNowEvents extends Yiisoft\ActiveRecord\Event\Handler\AttributeHandlerProvider` (идиома Yii3 AR: так же работает
`#[SoftDelete]`) → `getEventHandlers()` = `AfterInsert`, `BeforeUpdate`, `AfterUpdate`, `BeforeDelete`, `AfterDelete`
на `ObserverProvider::get()` (статический провайдер, как `ConnectionProvider` самого AR; без `set()` в bootstrap
обработчики пишут `warning` один раз и молчат). `EventsTrait` обязателен — без него AR событий не даёт (так устроен
`yiisoft/active-record`).

### Мост роутера

`YiiRouteUrlResolver` над `UrlGeneratorInterface::generateAbsolute($name, $args, [], null, $scheme, $host)`: в
HTTP-запросе host — из `CurrentRoute`, в консоли `generateAbsolute` без URI вернёт относительный путь — мост передаёт
scheme/host из `base_url`; `host:` правила → host параметром, scheme из `hosts.<host>.base_url` или `https`. Локаль —
аргумент `router.locale_parameter` (yii-demo: `_language`).

### Консоль, HTTP, dispatch

Команды — тонкие `symfony/console` обёртки над раннерами core (как в бандле; имена `indexnow:check`,
`indexnow:submit`, `indexnow:submit-record`, `indexnow:explain`, `indexnow:sitemap`, `indexnow:key:generate`).
`KeyFileHandler` — PSR-15 `RequestHandlerInterface` (yiisoft/router action) → `KeyFileResponder::bodyForKey($key,
$request->getUri()->getHost())`, `ResponseFactoryInterface` из контейнера. `dispatch`: `sync` (в `AfterEmit`),
`none`; `queue` не предлагается до релиза `yiisoft/queue` — приложение может подставить свой `DispatcherInterface`
(например, над `yiisoft/queue` dev или Cycle/Spiral очередью). Проверки: `ActiveRecordCheck` (`ObserverProvider`
установлен, `EventsTrait` у классов из `rules`), `RouterCheck` (маршрут ключа есть в `RouteCollectionInterface`),
`Check\SitemapSpoolCheck`.

## Тесты

Оба пакета гоняют три кита core: `CoreConformanceTestCase` (фасад из контейнера/компонента),
`OrmConformanceTestCase` A01–A21 (+A05b/A05c) через драйвер над AR (A20 — рецепт `updated_at`), HTTP-сценарии
H01–H06 (файл ключа 200/404/disabled, `check` exit-коды, отправка после ответа). Плюс специфика: verify-on-commit
(update откатан → не уходит; rename откатан → старый URL **не** объявляется удалённым), `models`-список без behavior
(Yii2), `link()` + рецепт (оба), команды через `SymfonyStyle` (Yii2: `StreamOutput` в `php://memory`).

Матрица CI: PHP 8.2–8.5 × (Yii2 2.0.45 lowest / latest; Yii3 highest / lowest), phpstan на highest. Yii2 требует
`yiisoft/yii2-composer` (allow-plugins) и bower-asset'ы — в dev через `yidas/yii2-composer-bower-skip`
(в приложениях пользователей asset-packagist уже настроен шаблоном).

## Открытое

- `yiisoft/queue` стабильный релиз → `dispatch: queue` для Yii3.
- Cycle ORM (`yiisoft/yii-cycle`, Spiral) — отдельный `indexnowkit/cycle` при спросе: два потребителя, свои
  события (`cycle/entity-behavior`), транзакции через `EntityManager` — кандидат на `TransactionStaging`.
- Yii2 без pretty URL: файл ключа доступен только как `?r=indexnow/key-file/index&key=` — движки его не найдут;
  `check` предупреждает, рецепт — статический файл в webroot (`key-generate --write-file=web/`), не реализовано.
