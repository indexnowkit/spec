# 13. PHP: `indexnowkit/laravel`

Laravel 11, 12, 13 (13 требует PHP 8.3). PHP ≥ 8.2. Тип `library`, auto-discovery через `extra.laravel.providers`
(`IndexNowKit\Laravel\IndexNowKitServiceProvider`). Зависимости: `indexnowkit/core ^0.2` и компоненты
`illuminate/{cache,console,contracts,database,queue,routing,support}`; `laravel/framework` целиком не требуется.
Тесты: PHPUnit 11 + `orchestra/testbench` (монорепо на PHPUnit, фреймворки не смешиваем — не Pest). Статический
анализ: phpstan level 9 + strict-rules монорепо, `larastan` поверх (расширение подключается в root `phpstan.neon.dist`,
в split-репо — в собственном). Стиль: php-cs-fixer монорепо (не Pint).

## Установка

```bash
composer require indexnowkit/laravel
php artisan vendor:publish --tag=indexnow-config      # config/indexnow.php
php artisan indexnow:key:generate --write-env         # INDEXNOW_KEY в .env
php artisan indexnow:check
```

`config/indexnow.php` = схема 02 (`Config::OPTIONS`) плюс Laravel-блоки ниже. Значения по умолчанию:
`key` из `env('INDEXNOW_KEY')`, `base_url` из `env('INDEXNOW_BASE_URL', env('APP_URL'))`, `environment` =
`app()->environment()` (в `production_environments` Laravel-имя `production` уже входит), `dispatch: queue`
(очередь есть в любом Laravel-приложении; `QUEUE_CONNECTION=sync` выполняет job inline).

## Дерево конфигурации

Ключи core (без изменений имён): `enabled`, `key`, `hosts`, `previous_key`, `key_location`, `base_url`, `strict_hosts`,
`environment`, `production_environments`, `max_url_length`, `engines`, `engine_aliases`, `locale_hosts`, `dispatch`
(`sync | queue | none`), `serve_key_file`, `dry_run`, `batch.max_urls`, `debounce.{per_url,key_prefix}`,
`throttle.max_requests_per_minute`, `http.{timeout,user_agent}`, `logging.{max_urls,forbidden_escalation,max_body,levels}`,
`retry.*`, `resolver.*`, `collector.*`.

Laravel-специфичные блоки (carve-out; `ConfigFactory::coreOptions()` вырезает их перед `Config::fromArray()`,
остаток проверяется `Config::unknownOptions()` — неизвестный ключ пишется `warning`, не ломает загрузку):

| Блок | Значение |
|---|---|
| `queue.connection` | connection очереди для `SubmitUrlsJob` (`null` = дефолт приложения) |
| `queue.queue` | имя очереди (`null` = дефолт connection) |
| `queue.delay` | секунд задержки перед первым выполнением (0) |
| `debounce.store` | `cache` (дефолтный `Cache::store()`), имя стора (`redis`), `memory`, `none` |
| `http.client` | id/класс PSR-18 клиента в контейнере (`null` = автообнаружение через `LazyTransport`) |
| `key_file.{enabled,path,host,cache_max_age,route_name,middleware}` | как в бандле; `path` содержит `{key}`, дефолт `/{key}.txt`; `middleware` — список для роута (по умолчанию пустой: без `web`, без сессии и CSRF) |
| `router.locales` | список локалей для `locales: 'all'` (пусто = только текущая) |
| `router.locale_parameter` | имя параметра маршрута для локали (`locale`); подставляется только если маршрут его объявляет |
| `router.set_app_locale` | `true`: на время генерации URL локали вызывается `App::setLocale()` и восстанавливается |
| `sitemap.{enabled,url,max_depth,max_sitemaps,max_bytes,allow_foreign_hosts,spool,spool_dir,fetch_retries}` | как в бандле |
| `logging.channel` | канал Laravel Log (`null` = дефолтный) |
| `eloquent.enabled` | `false` = observer не регистрируется даже у моделей с `IndexNowable` |

`retry.*` в Laravel **используется**: из `Config::retryPolicy()` job берёт `tries` и backoff (в бандле повторы делает
Messenger, здесь — сама job).

Проверки при загрузке (`ConfigFactory`, по образцу бандла): конфиг валидируется в рантайме, потому что значения
приходят из `env()`. `ConfigurationException` **не** бросается из observer'а или `terminating`: пишется `critical`
`indexnow: invalid configuration, IndexNow is disabled until it is fixed: ...` и используется
`Config(enabled: false, dryRun: true)`; точную ошибку печатает `indexnow:check`. `dispatch: queue` без `base_url`
— ошибка (воркер не имеет request-контекста).

## Объявление модели

```php
use IndexNowKit\Attribute\{IndexNow, IndexNowDefaults};
use IndexNowKit\Laravel\Eloquent\IndexNowable;

#[IndexNowDefaults(when: 'isPublished', fields: ['slug', 'title', 'body', 'published'])]
#[IndexNow(route: 'posts.show', params: ['post' => 'self'])]
#[IndexNow(route: 'posts.amp', params: ['post' => 'self'], when: 'hasAmp')]
#[IndexNow(via: 'category')]
class Post extends Model
{
    use IndexNowable;
}
```

Модель объявления общая (02): повторяемый `#[IndexNow]`, `#[IndexNowDefaults]`, `#[IndexNowUrl]` на методе;
всё компилируется `RuleCompiler` в `UrlRule[]`, дальше работают общие `ObjectChangeHandler`, `ChangeClassifier`,
`GuardedUrlResolver`. Адаптеру остаются мост к роутеру и хуки Eloquent.

- `params: ['post' => 'self']` передаёт модель в `route()` — route model binding через `getRouteKey()`
  (и `{post:slug}` binding field — Laravel подставляет поле сам).
- **Accessor'ы на Eloquent.** Атрибуты модели не являются PHP-свойствами, поэтому core не может прочитать `'slug'`
  рефлексией. Ядро даёт точку расширения `Attribute\ParamExtractor::registerReader(SubjectReaderInterface)`;
  адаптер регистрирует `EloquentSubjectReader`: для `Model` accessor читается через `getAttribute()`, если это
  атрибут, cast, mutator/accessor или отношение (метод с объявленным возвращаемым типом `Relation` либо уже
  загруженное); иначе accessor обрабатывает DSL core (метод `isPublished()`, `getX()`, свойство). Незнакомое имя
  → `ConfigurationException` (A10), а не `null`. `fields`/`whenFields` — имена атрибутов (колонок) как их пишет
  разработчик; конвенция `isPublished → published → is_published` работает.
- **Без trait/атрибута**: `IndexNowKit::observe(Post::class, [new IndexNow(...)], new IndexNowDefaults(...))`
  в `AppServiceProvider::boot()` регистрирует правила в `RuleRegistry` (тот же набор правил, не отдельный путь) и
  вешает observer на класс. `IndexNowKit::rules()` возвращает реестр для `registerFor()`.

Фасад Laravel — `IndexNowKit\Laravel\Facades\IndexNowKit` (не `IndexNow`: коллизия с атрибутом в импортах модели).
Корень фасада — `IndexNowKit\Laravel\IndexNowManager`: `submit()`, `submitModels(iterable)`, `submitModel()`,
`explain()`, `collect()`, `flush()`, `rules()`, `observe()`, `kit(): IndexNowKit\IndexNowKit`. Сам core-фасад
`IndexNowKit\IndexNowKit` тоже в контейнере (инъекция по типу).

## Хуки Eloquent и commit-safety

`IndexNowable::bootIndexNowable()` → `static::observe(IndexNowObserver::class)`; `#[ObservedBy]` пользователю не
нужен. Observer **синхронный** (не `ShouldHandleEventsAfterCommit`) и резолвит URL в момент события, пока живо старое
состояние; передача в `Collector` откладывается до реального COMMIT через `Connection::afterCommit()`.

Почему не `ShouldHandleEventsAfterCommit`: отложенный обработчик выполняется после `finishSave()`, где Eloquent уже
сделал `syncOriginal()` — `getOriginal()` равен текущему значению, change set (`[old, new]`) потерян. Классификация
`published → draft` вырождается в эвристику, `Equals`-условия дают ложные срабатывания, A21 (старый URL при смене
slug) невозможен, `deleting` сработал бы после удаления строки. Синхронный observer этих проблем не имеет, а
`DatabaseTransactionsManager` Laravel даёт нужную семантику: callback привязывается к самой глубокой открытой
транзакции, при `commit()` выполняется только когда уровень стал 0, при `rollBack()` до savepoint callback'и
вложенных уровней отбрасываются (A02, A05, A05b, A05c). Вне транзакции `afterCommit()` выполняет callback сразу.
`TransactionStaging` core не нужен; если у соединения нет менеджера транзакций (нестандартный `Connection`),
observer пишет `warning` и передаёт URL сразу.

События:

| Событие Eloquent | Действие |
|---|---|
| `created` | `changes()->created($model)` |
| `updated` | `changeSet = [field => [getOriginal(field), new]]` по `getChanges()`; `changes()->renamed($model, $changeSet, $previous)` (старые URL по прежним значениям, `$previous` = клон с `getRawOriginal()`), затем `changes()->updated($model, array_keys($changeSet), $changeSet)`; правило с `when: true → false` резолвится как `Deleted` |
| `deleting` | URL резолвятся (`changes()->deleted()`), пока строка и отношения живы, и запоминаются в `WeakMap` по модели |
| `deleted` | запомненные URL передаются (после `afterCommit`); при отменённом `deleting` (`false`) событие не приходит, запись WeakMap умирает с моделью |
| `restored` (SoftDeletes) | `Created` (страница снова публична); сопутствующий `updated` по `deleted_at` схлопывается дедупом коллектора |
| `forceDeleted` | как `deleted` (URL уже в WeakMap из `deleting`/`forceDeleting`) |

Soft delete = `Deleted` (страница отдаёт 404). Никакое исключение из observer'а не доходит до приложения:
`ObjectChangeHandler` и `GuardedUrlResolver` пишут `error` в лог; сам observer оборачивает передачу в `try/catch`.

Bulk-операции (A13): `Post::where(...)->update()`, `delete()`, `Model::insert()`, `upsert()`, `DB::table()` событий не
вызывают → документировано, ручной путь `IndexNowKit::submitModels(Post::whereIn('id', $ids)->get())` или
`php artisan indexnow:submit-model Post 1 2 3`. Изменение pivot (A20): `attach/detach/sync` событий владельца не
вызывают; рецепт — `$touches = ['posts']` на связанной модели, тогда `touch()` даёт `updated` владельца (правило
без `fields`-фильтра или с `updated_at` в `fields`).

## Collector и dispatch

- `Collector` — `$this->app->scoped()` (Octane сбрасывает между запросами).
- `flush()` в `app()->terminating()` (HTTP и artisan; Octane вызывает `terminate()` на каждый запрос) и на
  `Illuminate\Queue\Events\JobProcessed` (долгоживущий воркер, URL собранные внутри job).
- `dispatch: sync` → `SyncDispatcher` в `terminating`; `none` → `NullDispatcher`; `queue` → `QueueDispatcher`
  кладёт `SubmitUrlsJob` через `Bus` с `queue.connection/queue/delay`.
- `SubmitUrlsJob implements ShouldQueue`: `tries = retry.max_attempts`, `backoff()` из `RetryPolicy`
  (`base_delay × multiplier^n`, cap `max_delay`); после `submit()` — `Result::retryableUrls()`:
  ретраибельные → `RetryPolicy::delayAfter($results, attempts())` → `release($delay)` (429 → `Retry-After`),
  исчерпание попыток → `fail()`; 400/403/422 → `fail()` без retry (запись в `failed_jobs` — это оперативный
  сигнал «ключ не отдаётся»), лог `error`. Job хранит только `list<string> $urls`; `afterCommit` не нужен —
  observer уже после commit.
- Debounce: `Psr16DebounceStore` поверх `Cache::store(debounce.store)` (`Illuminate\Cache\Repository` реализует
  PSR-16), ключи `debounce.key_prefix`. `memory` для CLI-скриптов и тестов, `none` выключает.
- Throttle: `TokenBucket` core (`throttle.max_requests_per_minute`), фактически работает в воркере.

## Мост роутера

`LaravelRouteUrlResolver implements RouteUrlResolverInterface`:

- `locales('current')` → `[null]`; `'all'` → `router.locales`, пусто → `[null]`; список → как задан.
- `generate()`: `URL::route($name, $params, absolute: true)`; локаль подставляется параметром
  `router.locale_parameter`, **только если** маршрут объявляет такой параметр (иначе Laravel добавил бы query
  string); при `router.set_app_locale` на время генерации меняется `App::setLocale()`.
- Абсолютный root: внутри HTTP-запроса — как сгенерировал Laravel (текущий host); в консоли/воркере (`runningInConsole()`)
  — `base_url`; правило с `host:` — `hosts.<host>.base_url`, иначе `https://<host>`. Root подменяется в готовом URL
  (scheme/host/port), маршруты с `Route::domain()` сохраняют свой host. Глобальное состояние `UrlGenerator`
  (`forceRootUrl`) не трогается.
- `RouteNotFoundException` / `UrlGenerationException` (`InvalidArgumentException`) → `ConfigurationException` с
  именем маршрута.

`#[IndexNow(resolver: ...)]` резолвится `ContainerResolverLocator`: `app()->make($id)` для зарегистрированного
binding'а или автоматически разрешимого класса; не `UrlResolverInterface` или неразрешимо → `ConfigurationException`.

## Key file

Роут регистрируется провайдером: `Route::get(key_file.path, KeyFileController::class)->where('key', KeyValidator::PATTERN)
->name(key_file.route_name)->middleware(key_file.middleware)`, при `key_file.host` — `->domain()`. Без группы `web`:
без сессии и CSRF; совместимо с `route:cache` (контроллер-класс, не closure). Контроллер спрашивает
`KeyFileResponder::bodyForKey($key, $request->getHost())` → 200 `text/plain; charset=utf-8`,
`Cache-Control: public, max-age=key_file.cache_max_age`, `Vary: Host` при непустом `hosts`; иначе 404.
`key_file.enabled: false` (или устаревший `serve_key_file: false`) — роут не регистрируется (H03).

## Artisan

Имена и опции как в бандле:

- `indexnow:key:generate [--length=32] [--alphanumeric] [--write-env[=FILE]] [--force]` — `--write-env` пишет
  `INDEXNOW_KEY=` в `.env` (или FILE), идемпотентно; `--force` ротирует с предупреждением.
- `indexnow:check [--live] [--host=] [--probe-url=]` — `Checker` core + `CheckInterface` из контейнера
  (`$app->tag([...], 'indexnowkit.check')`): очередь (`queue.connection` существует; `sync` = без ретраев),
  cache-стор дебаунса разрешается, spool sitemap (`Spool::probeDisk`). Невалидный конфиг → exit 1 с точной ошибкой.
- `indexnow:submit <urls...> [-f|--force] [--dry-run] [--json]`.
- `indexnow:submit-model <model> [ids...] [--event=updated] [--limit=1000] [--explain] [-f|--force] [--dry-run] [--json]`
  — `<model>` FQCN или короткое имя в `App\Models\`; SoftDeletes-модели загружаются `withTrashed()` при `--event=deleted`.
- `indexnow:explain <model> <id> [--event=]` — правила → подписка → `when` → `fields` → URL → нормализация →
  host/ключ/файл ключа → дебаунс; ничего не отправляет.
- `indexnow:sitemap [sitemap] [--changed-since=] [--allow-foreign-hosts] [-f|--force] [--dry-run] [--json]` —
  потоково, порциями `batch.max_urls`, сводка `ResultSummary`; при обрыве недобранная порция отправляется до
  exit 1; зависит от `SitemapSourceInterface` (binding в контейнере, приложение может подменить).

Тела команд живут в core (`Console\*Runner` над `OutputStyle`, он же `SymfonyStyle`); artisan-команды только
разбирают ввод, слова («model», `php artisan`) задаёт binding `Console\Vocabulary`. `Console\ModelLoader` реализует
`Console\SubjectLoaderInterface` ядра. `--force`/`--dry-run` собирают отдельный `Submitter`
(`Console\SubmitterFactory`: `NullDebounceStore`, `Config::with(dryRun: true)`), не трогая сервис приложения.
Строка про observer в `indexnow:check` — `Check\EloquentCheck`, spool — `Check\SitemapSpoolCheck` ядра. Планировщик: `Schedule::command('indexnow:sitemap
--changed-since="1 day"')->daily()`.

## Контейнер

Binding'и по интерфейсам core (всё заменяемо через `$app->bind()` в приложении или `extend()`): `Config`,
`KeyProviderInterface`, `TransportInterface` (`LazyTransport`), `UrlNormalizerInterface`, `ThrottleInterface`,
`ClientInterface`, `DebounceStoreInterface`, `SubmitterInterface`, `CollectorInterface` (scoped),
`AttributeReaderInterface` (= `RuleRegistry` поверх `AttributeReader`), `RouteUrlResolverInterface`,
`ResolverLocatorInterface`, `UrlResolverInterface` (`AttributeUrlResolver`), `GuardedUrlResolver`,
`ObjectChangeHandler`, `DispatcherInterface`, `IndexNowKit\IndexNowKit`, `KeyFileResponder`, `CheckerInterface`,
`SitemapSourceInterface`, `IndexNowManager`. Логгер — `Log::channel(logging.channel)`; в тестах подменяется
`ArrayLogger`. Публикуемые теги: `indexnow-config`.

## Совместимость

Octane (scoped collector, `terminate()` на запрос), Vapor (роут вместо файла), Nova/Filament (обычные модели →
работает), Horizon (обычная job). Laravel Scout не связан. `laravel-freelancer-nl/laravel-index-now` и
`ymigval/laravel-indexnow` не имеют post-commit и автотриггера; `nomadicsoft/laravel-indexnow` ближайший по функциям,
без атрибута, маршрутного резолвера и общего core.

## Тесты

Testbench, sqlite in-memory, схема через `Schema::create` в setUp, роуты через `defineRoutes`; транспорт —
`IndexNowKit\Testing\FakeTransport` (`$app->instance(TransportInterface::class, ...)`), логгер — `ArrayLogger`;
`Http::fake()` не используется (PSR-18).

- `CoreConformanceTestCase` ядра против фасада из контейнера (C01, C03, C04, C06, C09–C12, C14, C19, C20).
- `OrmConformanceTestCase` ядра (03, «Тест-кит для адаптеров»): A01–A21 (+A05b/A05c) через драйвер над Eloquent
  (create/update/delete/begin/commit/rollback/flush и фикстуры Post/MultiPost/CategorizedPost/…); A20 через `$touches`.
- H01–H06: key file (200/404/disabled), `indexnow:check` exit-коды, POST после ответа (`terminating` в
  `$this->get()` Testbench).
- Laravel-специфика: dispatch `queue` (`Queue::fake()` → job в очереди; job с 429 → `release`, 403 → `fail`),
  SoftDeletes, `observe()` без trait, `ConfigFactory` с невалидным env, `--json` команд, `route:cache`-совместимость
  роута ключа.

Матрица CI: PHP 8.2–8.5 × Laravel 11 (`--with laravel/framework:^11.0` + testbench 9, `policy.advisories.block false`:
все релизы L11 под security advisories), 12 (root platform php 8.2) и 13 (job с platform 8.4 + testbench 11); phpstan+larastan
на highest.
