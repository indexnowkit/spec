# 13. PHP: `indexnowkit/laravel`

Laravel 11, 12. PHP ≥ 8.2. Auto-discovery `extra.laravel.providers`. Тесты Pest + Testbench,
Larastan, Pint.

## Установка

```bash
composer require indexnowkit/laravel
php artisan vendor:publish --tag=indexnow-config   # config/indexnow.php
php artisan indexnow:key --write-env
```

`config/indexnow.php` = схема 02, значения из `env('INDEXNOW_KEY')`, `env('APP_URL')` как
дефолт `base_url`, `dispatch` дефолт `queue` (Laravel всегда имеет очередь; `QUEUE_CONNECTION=sync`
работает inline).

## Объявление модели

```php
#[IndexNow(route: 'posts.show', params: ['post' => 'self'], when: 'isPublished', fields: ['slug','title','body','status'])]
class Post extends Model { use IndexNowable; }
```

- Атрибут из core. `params: ['post' => 'self']` передаёт модель в `route()` (route model
  binding через `getRouteKey()`). Без атрибута trait ищет `indexNowUrls(): array`.
- Trait `IndexNowable` регистрирует observer в `bootIndexNowable()` (Eloquent boot-конвенция),
  атрибут `#[ObservedBy]` не нужен пользователю.
- Альтернатива без trait: `IndexNow::observe(Post::class, url: fn (Post $p) => route('posts.show', $p))` в `AppServiceProvider::boot`.

## Commit-safety

`IndexNowObserver implements ShouldHandleEventsAfterCommit`: события `created`, `updated`,
`deleting`, `deleted` выполняются после commit внешней транзакции, немедленно вне транзакции.
Это единственная корректная точка в Laravel (модельные события иначе синхронны и не знают о
rollback).

- `updated`: `wasChanged($fields)` для `fields`; `when` до/после через `getOriginal($field)`
  для перехода published → draft (`Deleted`).
- `deleting` (до удаления, но после commit? нет: `deleting` внутри транзакции): URL считаем в
  `deleting` и кладём в `$model->indexNowPendingUrls` (свойство на модели, не persisted);
  `deleted` (after commit) отправляет. `SoftDeletes`: `deleted` при soft delete тоже
  срабатывает — трактуем как Deleted; `restored` как Created.
- Mass `update()/delete()` без событий (A13): `IndexNow::submitModels($query->get())`.

## Collector и dispatch

- Request-scoped singleton `Collector` (`$this->app->scoped(...)`, сбрасывается Octane).
- `app()->terminating(fn () => $collector->flush())` для HTTP и CLI.
- `dispatch: queue`: `SubmitUrlsJob implements ShouldQueue` с `$tries = 4`, `backoff()` по 01
  (`[60, 300, 900]`), `afterCommit` не нужен (наблюдатель уже после commit), queue name из
  конфига. 429 → `release($delay)`; 403/422 → `fail()` без retry (лог).
- `dispatch: sync`: в `terminating`.
- Debounce store: `Cache::store(config('indexnow.debounce.store'))` через `Psr16DebounceStore`
  (`Illuminate\Cache\Repository` реализует PSR-16).

## Key file

Route в `routes/indexnow.php`, загружается провайдером: `Route::get('/{key}.txt', KeyFileController::class)->where('key', '[A-Za-z0-9-]{8,128}')`,
без middleware `web` (без сессии/CSRF). `serve_key_file: false` отключает.

## Artisan

`indexnow:key [--write-env]`, `indexnow:check`, `indexnow:submit {url*}`, `indexnow:submit-model {model} {id*}`,
`indexnow:sitemap {url?}` (без url: если установлен `spatie/laravel-sitemap`, обходит его генератор).
Планировщик: README-рецепт `Schedule::command('indexnow:sitemap --changed-since=1d')->daily()`.

## Совместимость

Octane (сброс scoped), Vapor (роут вместо файла), Nova/Filament (обычные модели → работает).
Laravel Scout не связан. `laravel-freelancer-nl/laravel-index-now` и `ymigval/laravel-indexnow`
не имеют post-commit и автотриггера; `nomadicsoft/laravel-indexnow` (76 загрузок) ближайший по
функциям, у него нет атрибута/маршрутного резолвера и общего core.

## Тесты

Testbench, sqlite, `Http::fake()` не используем (PSR-18 клиент), mock-сервер. A01–A14, H01–H06.
