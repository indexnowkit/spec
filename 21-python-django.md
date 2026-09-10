# 21. Python: `indexnowkit-django` (волна P, шаг 2)

Статус: **переписана 2026-09-10** (прежняя редакция 2026-09-03 заменена). Базируется на спеке 20; эталон поведения —
`indexnowkit/laravel` (observer + after-commit фреймворка, спека 13) и `indexnowkit/symfony-bundle` (команды, checks). Решения — §9.

## 0. Цель и границы

`pip install indexnowkit-django`, `INSTALLED_APPS += ["indexnowkit_django"]`, один блок `INDEXNOW = {...}` в settings, одна
строка в `urls.py`, `@indexnow(...)` или `get_absolute_url()` на модели — и страницы уходят в IndexNow после commit, дебаунсом,
батчами, с `manage.py indexnow_check`. ORM-хуки через сигналы, commit-safety через `transaction.on_commit` (нативно — сводка
`docs/spec/README.md`), management commands с теми же именами, что `indexnow:*` у PHP, `checks` framework с кодами `check`,
key-view, `django.contrib.sitemaps` как источник `sitemap`, `django.tasks` как dispatch. Границы: Wagtail — спека 25; DRF —
ничего специального (сигналы те же); django-cms — по спросу.

## 1. Факты (проверены 2026-09-10, docs.djangoproject.com/en/6.1 если не сказано иначе)

1. **Версии** (спека 20 §1.1): 5.2.17 LTS (Python 3.10–3.14, до 2028-04), 6.0.8 (3.12–3.14, до 2027-04), 6.1.1 (3.12–3.14, до
   2027-12); 4.2 и 5.1 вне поддержки. Классификаторы `Framework :: Django :: 5.2 | 6.0 | 6.1` есть. pytest-django 4.14.0 требует
   `django>=5.2`. django-stubs 6.1.0 (2026-08-12, ≥3.11).
2. **`transaction.on_commit(func, using=None, robust=False)`** (topics/db/transactions «Performing actions after commit»): колбэк
   выполняется после commit **самого внешнего** `atomic`; в autocommit — сразу; при rollback внешнего блока — отбрасывается; при
   откате savepoint отбрасываются колбэки, зарегистрированные внутри него (документированный пример `foo()` вызван, `bar()` — нет);
   `robust=True` — исключение колбэка логируется в `django.db.backends.base`, следующие колбэки выполняются; в `TestCase` колбэки
   не выполняются — `captureOnCommitCallbacks(execute=True)`; `TransactionTestCase` выполняет.
3. **Сигналы** (ref/signals): `pre_save`/`post_save(sender, instance, created, raw, using, update_fields)`;
   `pre_delete`/`post_delete(sender, instance, using, origin)` — `origin` (модель или QuerySet, откуда пошло удаление) с 6.0;
   `m2m_changed(sender=through, instance, action pre_/post_add|remove|clear, reverse, model, pk_set, using, raw)` (`raw` — 6.1);
   `QuerySet.update()`, `bulk_create()`, `bulk_update()`, `QuerySet.delete()` (для каждого объекта — шлёт `pre/post_delete` только
   при Python-каскаде; `DB_CASCADE` 6.1 — не шлёт) сигналов сохранения **не шлют**; `request_finished` — «Sent when Django
   finishes delivering an HTTP response to the client» (шлётся из `HttpResponseBase.close()`, который WSGI-сервер зовёт после
   отдачи тела — «после ответа», H06); `request_started`.
4. **Отслеживание полей**: `Model.from_db(cls, db, field_names, values, *, fetch_mode=None)` — документированный паттерн
   `instance._loaded_values = dict(zip(field_names, values))` для сравнения в `save()` (ref/models/instances); `save(update_fields=…)`
   даёт множество имён; `refresh_from_db`; `FETCH_ONE/FETCH_PEERS/FETCH_RAISE` (6.1) — `_loaded_values` не покрывает
   отложенные поля (`DEFERRED`). `django-model-utils` `FieldTracker` — пакет 5.0.0 от 2024-09-04, не нужен.
5. **`django.tasks`** (6.0; topics/tasks, ref/tasks): `@task(priority, queue_name, backend, takes_context, **kwargs в 6.1)`,
   `task.enqueue(*args, **kwargs)` / `aenqueue`, аргументы и результат — JSON (`datetime`, модель — `TypeError`; кортеж → список);
   `TASKS = {"default": {"BACKEND": "…"}}`; в поставке только `ImmediateBackend` (синхронно при enqueue) и `DummyBackend` (не
   выполняет); документированный рецепт «после commit» — `transaction.on_commit(partial(task.enqueue, …))`; production-бэкенды —
   сторонние: `django-tasks-db` 0.13.0 (ORM), `django-tasks-rq` 0.12.0, `huey` (djangoproject.com/community/ecosystem «Tasks»);
   бэкпорт `django-tasks` 0.12.0 для Django ≥4.2. Ретраев во фреймворке нет (docs не упоминают).
6. **Checks** (topics/checks): `@register(Tags.x, deploy=False)` в `AppConfig.ready()`, сообщения `Error/Warning/Info` с
   `id="indexnowkit.E001"`, `manage.py check --tag indexnow`, `--deploy`.
7. **Management commands** (howto/custom-management-commands): имя = имя модуля в `management/commands/` (без двоеточий —
   `indexnow_check`), `add_arguments(parser)` — argparse, `self.stdout`/`self.style`, `CommandError(returncode=…)`, `requires_system_checks`.
8. **Кэш** (topics/cache): `cache.add(key, value, timeout)` — атомарно на memcached и Redis (на остальных — get+set), `get_many`/
   `set_many`, `touch`, `timeout=None` вечно; ключ ≤ 250 символов без пробелов (`CacheKeyWarning`), `caches["alias"]`, экземпляр на поток.
9. **Sitemaps** (ref/contrib/sitemaps): `Sitemap.items()`, `location(item)` (иначе `get_absolute_url()`), `lastmod(item)`,
   `get_urls()`, `protocol`, словарь `sitemaps` во view; `ping_google` в документации 6.1 отсутствует.
10. **Async**: `transaction.on_commit` — синхронный API; в async view запись через `sync_to_async(thread_sensitive=True)` — сигналы и
    `on_commit` срабатывают в том потоке; `contextvars` через `sync_to_async` копируются (asgiref). Проверить при реализации (§7.1).
11. **Конкуренты**: `django-indexnow` (hckjck, GitHub, не на PyPI, 0★, 2026-03): сигналы `post_save`/`post_delete`, дедуп 60 с в
    кэше, middleware ключ-файла в корне, stdlib-only; без `on_commit`, батчей, `check`, sitemap. `wagtail-indexnow` — спека 25.

## 2. Принципы

1. **Сигналы — только для зарегистрированных моделей** (`sender=Model`), не глобально; ничего не отправляется, пока нет ни одного
   правила и ключа (спека 00 п. 5).
2. **Commit-safety — `on_commit`, ничего своего.** Резолв синхронно в сигнале (старое состояние живо), доставка в
   `on_commit(robust=True, using=using)`; никакого `TransactionStaging`.
3. **Настройки — та же схема.** `INDEXNOW = {...}` в **snake_case, вложенно** — ровно `Config.from_mapping()` спеки 20 плюс блоки
   адаптера; `INDEXNOW_*` переменные окружения тем же правилом поверх (решение §9.1).
4. **Команды и коды — как у семейства.** `manage.py indexnow_<cmd>` с `Definitions` core; `check` — те же коды; система `checks`
   Django — зеркало `check` на `manage.py check`/старте.
5. **Хуки never-throw**, `ConfigFactory.load()` — `critical` + disabled, не падение при старте (кроме `check`).
6. **Ни одной зависимости кроме `indexnowkit` и `Django>=5.2`.**

## 3. Дизайн

### 3.1. Пакет

`python/packages/indexnowkit-django/`, модуль `indexnowkit_django`, `dependencies = ["indexnowkit>=0.1,<0.2", "Django>=5.2"]`,
`requires-python >=3.11` (Django 5.2 на 3.11 — законная пара до 2028-04; 6.x сам потребует 3.12). `AppConfig` `IndexNowKitConfig`
(`name = "indexnowkit_django"`, `label = "indexnowkit"`, `default_auto_field`): `ready()` — конфигурация через `ConfigFactory.load()`,
регистрация сигналов для моделей с правилами, регистрация checks, autodiscovery `indexnow.py` в приложениях (`registry.register()`
для чужих моделей — `django.utils.module_loading.autodiscover_modules("indexnow")`).

Классы (≤ 300 строк каждый): `apps.IndexNowKitConfig`, `conf.Settings` (чтение `settings.INDEXNOW` + `mapping_from_env()` поверх,
`environment` из `INDEXNOW_ENV`/`DJANGO_ENV`/иначе `"dev" if settings.DEBUG else "production"`? — нет: `environment` берётся из
`INDEXNOW["environment"]`, иначе `INDEXNOW_ENV`, иначе `None`; страховка `dry_run` тогда по `DEBUG`: `settings.DEBUG` и нет ключа →
`dry_run` (документировать как «Django: DEBUG заменяет environment, когда он не задан»)), `wiring.Services` (`ServicesBuilder` слоя 2:
`debounce_store` → `CacheDebounceStore(caches[alias])`, `router` → `DjangoRouteUrlResolver` (`reverse`, `translation.override(locale)`,
`Site`/`base_url`), `resolver_locator` → dotted path (`import_string`), `queue_factory` → §3.5, `submission_store` → §3.8,
`param_extractor` → `DjangoSubjectReader`), `hooks.Observer` (§3.3), `views.key_file`, `urls`, `checks`, `management/commands/*`,
`sitemaps.DjangoSitemapSource`, `tasks.py` (если `django.tasks` доступен), `signals.py` (`indexnow_submitted`, `indexnow_failed` —
Django-сигналы над `add_listener`), `models.py` + `migrations/0001` (история, §3.8).

### 3.2. Объявление модели

```python
from indexnowkit import indexnow, indexnow_defaults
from indexnowkit_django import IndexNowModelMixin

@indexnow_defaults(when="is_published", fields=["slug", "title", "status"])
@indexnow(route="blog:post_detail", params={"slug": "slug"})
class Post(IndexNowModelMixin, models.Model):
    ...

class Article(IndexNowModelMixin, models.Model):          # без декораторов: правило `url` из get_absolute_url()
    def get_absolute_url(self): return reverse("article", args=[self.slug])

# indexnow.py любого приложения — чужие модели
from indexnowkit import registry
registry.register(ThirdPartyModel, rules=[indexnow(url="get_absolute_url", when="published")])
```

- Один и тот же `UrlRule` спеки 02: `route` — имя маршрута с namespace, `params` — accessor'ы к полям/свойствам (`"category.slug"` —
  `select_related` не делается, обращение к FK — запрос; `explain` это показывает); `locales: "all"` — `settings.LANGUAGES`
  (`router.locales` заполняется автоматически, переопределяемо), генерация под `translation.override(locale)`, `i18n_patterns`
  совместимы; `host` — `Site`/`hosts`.
- `IndexNowModelMixin`: (a) `from_db()` кладёт `_indexnow_loaded` (снимок полей, факт §1.4) — старое состояние для `ChangeClassifier`
  без запроса; (b) правило `url` из `get_absolute_url()` по умолчанию, если декораторов нет. Миксин **не обязателен**: модель без
  него с `@indexnow` работает, старое состояние тогда берётся одним запросом в `pre_save` (`Model._base_manager.using(using).filter(pk=…)
  .values(*needed)`) только для полей из `fields`/`when_fields`/параметров правил (решение §9.2).
- Имена полей — `Field.name` модели (`update_fields` их и даёт), не колонки.
- `fields` не задан → любое сохранение = `updated` (как PHP).

### 3.3. Хуки и commit-safety

`hooks.Observer` над `adapter.ObserverHelper`:

- `pre_save(sender, instance, raw, using, update_fields)`: `raw` → ничего (загрузка фикстур); если нет `_indexnow_loaded` и `pk`
  задан — снимок запросом (§3.2); вычислить `change_set` (`{field: (old, new)}`) по снимку; `changes.updated_events(instance, changed, change_set)` /
  `created_events` (`_state.adding`) — **резолв до записи** для событий `deleted` (когда `when` стал ложным) и `renamed()` (старый
  slug); сложить в `instance._indexnow_pending`.
- `post_save(sender, instance, created, …)`: резолв оставшихся событий (теперь есть `pk`), объединить с pending, обновить снимок,
  `transaction.on_commit(partial(deliver, urls), using=using, robust=True)`.
- `pre_delete`: `changes.deleted(instance)` → `remember_deletion(instance, urls)`; `post_delete`: `take_deletion` → `on_commit`.
  `origin` (6.0) — не используется для решения (каждый объект каскада шлёт свои URL; это правильно).
- `m2m_changed` (`post_add/post_remove/post_clear`, `reverse` — владелец другой): владелец коллекции — `changes.updated(owner, [field])`
  (A20; поле — имя M2M-поля).
- `deliver(urls)` = `kit.collect(urls)`: внутри HTTP-запроса — в scope коллектора (flush на `request_finished`); вне запроса
  (`manage.py`, задача Celery, shell) — scope нет → сразу диспетчер (`Collector.collect` без scope, спека 20 §3.7). Итог: A01–A06:
  A02/A05/A05c — семантика savepoint факта §1.2; A05b/A06 в запросе — один POST, потому что колбэки внутренних `atomic` выполняются
  на внешнем commit и `collect` дедупит в одном scope. Вне запроса каждый `on_commit`-колбэк — свой `collect` без scope, то есть
  свой POST (три объекта в одной транзакции shell-скрипта — три POST): колбэки независимы, буфер «между колбэками одной транзакции»
  у Django нет. Принято и документируется («в shell/Celery оборачивайте bulk-работу в `with indexnow.collecting():`» — тогда
  один POST); кит A05b/A06 гоняется в scope (§5), риск §7.2.
- Сигналы подключаются **на модели с правилами** после `ready()`; для моделей, зарегистрированных позже (`registry.register` из
  `ready()` другого приложения) — реестр эмитит событие, адаптер доподключает.
- `request_started` → `collector.reset()` (warning о потерянных — как везде); `request_finished` → `kit.flush()` (после ответа под
  WSGI и ASGI, факт §1.3). Альтернатива middleware — до отправки ответа; не используется (H06).

### 3.4. Bulk и ручная отправка (A13)

`QuerySet.update/delete`, `bulk_create/update`, `DB_CASCADE`, raw SQL — сигналов нет. Адаптер даёт `indexnowkit_django.submit_queryset(qs,
event="updated")` (итерирует `.iterator(chunk_size=…)`, `changes.<event>`, чанки в диспетчер), `manage.py indexnow_submit_models
app.Model [ids…] --event`, и в README — раздел «Bulk».

### 3.5. Dispatch

| `dispatch` | Что | Условия |
|---|---|---|
| `auto` (дефолт адаптера) | `tasks`, если `settings.TASKS["default"]["BACKEND"]` задан и не `Immediate`/`Dummy`; иначе `sync` | `auto_dispatch` в `ConfigFactory` |
| `sync` | после ответа (`request_finished`) | воркер занят ≤ `http_timeout`; README советует `http.timeout: 3` для sync |
| `tasks` | `django.tasks` (6.0+) или `django_tasks` (бэкпорт на 5.2): `@task(queue_name=<queue.name>)` `submit_urls(urls, attempt)`; в воркере `WorkerOutcome`, ретрай — повторный `enqueue` через `on_commit`? Нет — ретрай с задержкой: у `django.tasks` нет `countdown` → `ThreadDispatcher`-подобное ожидание в задаче недопустимо; **ретрай = повторная задача без задержки не более `retry.max_attempts`, `Retry-After` уважать через `time.sleep` ≤ `retry.max_delay`? Нет** — §7.3: ретраи для `tasks` только если бэкенд `supports_defer` (`run_after`); иначе одна попытка + warning в `check` (`queue.driver`) | `need_base_url` |
| `callable` | dotted path `fn(urls: list[str]) -> None` — Celery `@shared_task` (`apply_async(countdown=delay)` в README), RQ, Dramatiq, huey | `need_base_url` |
| `thread`, `none` | из core | — |

Блок адаптера: `queue: {name: "default", backend: "default"}` (для `tasks`), `callable: "myapp.tasks.submit_indexnow"`.

### 3.6. Ключ-файл

`indexnowkit_django.urls`: `re_path(r"^(?P<key>[A-Za-z0-9-]{8,128})\.txt$", key_file, name="indexnowkit-key-file")`; view — `body_for_key(key,
request.get_host()-без-порта)` → `HttpResponse(body, content_type=CONTENT_TYPE)` с `config.key_file_headers()`, иначе `Http404`; без
сессии/CSRF (`@csrf_exempt` не нужен для GET). `key_file.path` не нужен (в `urls.py` пользователь ставит префикс сам); `key_file.enabled:
false` → 404. `previous_key` во время ротации.

### 3.7. Команды

`management/commands/`: `indexnow_check`, `indexnow_config`, `indexnow_submit`, `indexnow_submit_models` (`model` = `app_label.ModelName`
или короткое имя через `apps.get_model`, `ids…`, `--event`, `--limit`, `--explain`, `--force`, `--dry-run`, `--json`), `indexnow_explain`,
`indexnow_key_generate` (`--write-env` → `.env` в `BASE_DIR`), `indexnow_sitemap` (§3.9), `indexnow_history`, `indexnow_status`.
Каждая — `BaseCommand` с `add_arguments` из `Definitions.<cmd>().apply_to_argparse(parser)` и `handle()` → раннер core с
`Io(self.stdout, self.stderr)`; `requires_system_checks = []` у `indexnow_check` (сам проверяет). `Vocabulary(subject="model",
subjects="models", cli="python manage.py", submit_subjects="indexnow_submit_models", check="indexnow_check", …)`.
`ConfigSource` → `DjangoConfigSource` (`raw()` = `settings.INDEXNOW` + env, `build()` = `ConfigFactory.build`, `packages()`).

### 3.8. История, дебаунс, кэш

- `debounce.store`: `cache` (дефолт адаптера — `caches["default"]`) или alias из `CACHES`; `memory`, `none`; `CacheDebounceStore`
  использует `add()` для атомарной пометки (факт §1.8), ключи `<prefix><sha1(url)>` ≤ 250 символов. `DebounceStoreCheck` с пробой
  `set/get` `PROBE_KEY`; `is_shared()` — 403-счётчик и robots-кэш в том же кэше.
- `history.store`: `null` (дефолт), `sqlite` (core), `django` — модель `IndexNowSubmission` (`app_label = "indexnowkit"`, миграция
  `0001`; таблица `indexnowkit_submission`; поля как `SubmissionRecord`: `at`, `engine`, `host`, `status`, `reason`, `http_code`,
  `retryable`, `urls` JSONField, `url_count`; индексы `at`, `host`) — `DjangoSubmissionStore` реализует `HistoryStore` (`purge`, `count`,
  `last`), S01–S08 через `SubmissionStoreConformance`; `history.django.database` — alias БД.
- `check` строки адаптера: `wiring.signals` (модели с хуками: N), `queue.backend`/`queue.driver` (Immediate/Dummy — warning), `key_file.route`
  (маршрут `indexnowkit-key-file` в `urlconf` — `reverse` ok / error), `router.locales`, `debounce.store`, `history.store`, `settings.debug`
  (DEBUG=True в production-окружении — warning), `sitemap.django` (число классов `Sitemap` найдено).

### 3.9. Sitemap

`indexnow_sitemap` — два источника: URL/файл (ридер core) и `--from-django-sitemaps` (`DjangoSitemapSource`: словарь `sitemaps` из
`settings.INDEXNOW["sitemap"]["django"] = "myproject.urls.sitemaps"` или найденный по `resolve("django.contrib.sitemaps.views.sitemap")`
kwargs — decision: явный dotted path, автопоиск — warning в `check`); `lastmod(item)` → `--changed-since`, `--new-only` через
`SeenStore` (`sqlite`-файл `BASE_DIR/.indexnow/state.sqlite` или модель `IndexNowSeen` — та же миграция). Рецепт README: `Sitemap.lastmod`
из `updated_at`, чтобы Google получал сигнал из sitemap.

### 3.10. Checks framework

`checks.py` (`Tags.indexnow = "indexnow"`): `indexnowkit.E001` конфигурация не строится (текст `ConfigurationError`), `E002` `key_file.enabled`
без маршрута `indexnowkit-key-file`, `E003` `dispatch` требует `base_url`, `W001` `dispatch: sync` при `DEBUG=False` без `tasks`/`callable`
(совет), `W002` модель с правилом `route` на несуществующий маршрут (`NoReverseMatch` на фиктивных параметрах — нет, только имя
через `get_resolver().reverse_dict` — `check` печатает), `W003` `LANGUAGES` пуст при `locales: "all"`, `W004` `TASKS` backend Immediate/Dummy
при `dispatch: tasks`. Это подмножество `indexnow_check` для `manage.py check`/`runserver`; полный — команда.

### 3.11. Профилирование и наблюдаемость

Django-сигналы `indexnow_submitted(sender=Result)`/`indexnow_failed` над `add_listener` — для метрик пользователя. Панель
`django-debug-toolbar` — не в волне (спека 26 §8). Логгер `indexnowkit.*` — в `LOGGING` пользователя; README даёт блок.

## 4. BC и версии

`indexnowkit-django` 0.1.0; `Django>=5.2,<6.2`; тестируется 5.2 / 6.0 / 6.1 на 3.11 (5.2 только) … 3.14. Публичный контракт — настройки,
команды, `IndexNowModelMixin`, `submit_queryset`, сигналы, view; классы `wiring` — internal до 1.0 (`docs/bc.md`).

## 5. Тесты

- pytest-django; тестовый проект `tests/project/` (`settings.py`, приложение `blog` с фикстурами кита: post, multi-post, categorized post
  с `category` FK и `tags` M2M, category, tag, untracked, broken, bad-decorator); sqlite in-memory; **`OrmConformance`-драйвер** —
  A01–A21 (+A05b/A05c/A10b): `begin()` = `transaction.atomic().__enter__()` в стеке, `commit()` = `__exit__(None…)`, `rollback()` = `set_rollback(True)`
  + выход; `flush()` = `kit.flush()`; всё под `TransactionTestCase`-семантикой (`django_db(transaction=True)`) — иначе `on_commit` не
  сработает (факт §1.2); альтернатива `captureOnCommitCallbacks` — второй тест на A01 без transaction=True.
- `CoreConformance` над фасадом из `Services`; H01–H06 (`test_http.py`: `client.get("/<key>.txt")` → `KeyFileAssertions`; `call_command("indexnow_check")`
  → `CheckOutputAssertions`; H06 — `request_finished` после `response.close()`: `django.test.Client` зовёт `close()`).
- `test_dispatch_tasks.py` под `pytest.importorskip("django.tasks")`/`django_tasks`; `test_dispatch_callable.py`; `test_checks.py`;
  `test_commands.py` (девять команд, `--json`); `test_sitemaps.py`; `test_history_django.py` (S01–S08 + `purge`); `test_readme.py`.
- Матрица CI: Django 5.2 × (3.11, 3.12, 3.13, 3.14), 6.0 × (3.12, 3.14), 6.1 × (3.12, 3.13, 3.14); `lowest` — Django 5.2.0 на 3.11;
  PostgreSQL job для `history.django` (как `history-databases` PHP).

## 6. Документация

README EN/RU по шаблону (settings-блок 8 строк, `urls.py` одна строка, модель, `manage.py indexnow_check`, «How it works» с `on_commit`,
Bulk, Tasks/Celery, multisite `Site`+`hosts`, Limitations: `QuerySet.update`, `DB_CASCADE`, `raw`; Other packages; AI-notes);
`docs/`: `configuration.md` (блок, env, dispatch), `models.md` (декораторы ↔ поля Django, `get_absolute_url`, FK в `params`, i18n),
`tasks.md` (django.tasks, Celery, RQ рецепты), `commands.md`, `checks.md`, `sitemaps.md`, `multi-site.md`, `testing.md`
(`captureOnCommitCallbacks`, `FakeTransport` binding), `troubleshooting.md`, `bc.md`.

## 7. Риски и что проверить первым

1. **Async views**: `sync_to_async` + `on_commit` + contextvars — проверить, что scope коллектора, открытый в `request_started`
   (шлётся из потока обработчика ASGI? — `ASGIHandler` шлёт `request_started` в `sync_to_async`), виден в `on_commit`. Если нет —
   `CollectorMiddleware` ASGI из core поверх `ASGIHandler` в README.
2. **A05b вне запроса** (§3.3): два `on_commit` = два `collect` без scope = два POST. Решение: `deliver` вне scope открывает
   scope-на-`on_commit`-пакет невозможно; принять и документировать («в shell/Celery каждая транзакция — своя отправка»).
   Проверить, что в `TestCase`-подобном сценарии с `captureOnCommitCallbacks(execute=True)` A05b даёт один POST в scope.
3. **`django.tasks` без отложенного запуска**: `run_after` есть у `TaskResult`/`enqueue`? Проверить в ref/tasks (`supports_defer`
   в таблице бэкендов — есть: `ImmediateBackend` нет, `DummyBackend` да) — значит `enqueue(run_after=…)`? Уточнить API до кода;
   без него ретрай 429 — только `callable`-режим (Celery `countdown`).
4. **`from_db` и `.only()`/`defer()`**: снимок без отложенных полей → `when_fields` из отложенных не сравниваются → считать «изменилось»
   (правило спеки 02: невычислимое старое `when` при изменённом поле = переключение).
5. **`origin` каскадов**: `Post.delete()` каскадит `Comment` — `pre_delete` для каждого; `via: "post"` у комментария при удалении
   поста даст `updated` поста, который сам уходит как `deleted` — дедуп в scope, `deleted` побеждает? Порядок в POST не важен
   (один URL); `explain` покажет оба. Тест.
6. **`request_finished` при streaming-ответах** — шлётся при `close()` после итерации; ok. `FileResponse` — то же.
7. **Транзакции нескольких БД** (`using`): `on_commit(using=using)` — колбэк на той БД; тест с `databases = {"default", "other"}`.
8. **`settings.DEBUG` как environment**: `DEBUG=True` в проде с ключом — `check` warning (`settings.debug`), не dry-run.

## 8. Не делать (рассмотрено)

- `ShouldHandleEventsAfterCommit`-подобная отложенная классификация после commit (Laravel-урок: старые значения потеряны).
- `django-model-utils` `FieldTracker` — не нужен при `from_db`-снимке.
- Middleware ключ-файла в корне (как `django-indexnow`): маршрут в `urls.py` — явный, тестируемый, без перехвата каждого запроса.
- Ключ из `SECRET_KEY` (как `wagtail-indexnow`): ключ — `INDEXNOW_KEY`, ротация независима от `SECRET_KEY`.
- Панель debug-toolbar, admin-страница истории — по спросу.
- Поддержка Django 4.2/5.1 — вне поддержки апстрима.

## 9. `[решение]` — рекомендации

1. **`INDEXNOW = {...}` в snake_case, вложенно (форма `Config.from_mapping`)** — да; альтернатива UPPER-ключи по конвенции Django —
   вторая схема и второй генератор доков ради стиля; env-переменные `INDEXNOW_*` — единственная «верхняя» форма.
2. **Старое состояние: `from_db`-снимок в миксине, запрос в `pre_save` без миксина** — да; альтернатива «всегда запрос» — +1 запрос
   на каждое сохранение зарегистрированной модели.
3. **`dispatch: auto` (tasks при настроенном production-бэкенде, иначе sync)** — да; альтернатива `sync` по умолчанию как у core —
   пользователь с `django-tasks-db` не заметит, что ничего не ушло в очередь.
4. **`history.store: django` — модель с миграцией в приложении** — да (миграция всё равно нужна для `SeenStore`); альтернатива —
   только `sqlite`-файл: неудобно в контейнерах.
5. **Команды `indexnow_<cmd>` (подчёркивания), `indexnow_submit_models` с аргументом `model`** — да (двоеточие невозможно).
6. **Wagtail — отдельный пакет 25, не часть django-адаптера** — да.
7. **`--from-django-sitemaps` через явный dotted path словаря `sitemaps`** — да; автопоиск по `urlconf` — только предупреждение.

## 10. Definition of Done

- Чистый проект `django-admin startproject` + `pip install indexnowkit-django`: 8 строк settings, 1 строка urls, `@indexnow` на модели →
  `manage.py indexnow_check` зелёный против мок-сервера, `manage.py check --tag indexnow` без ошибок, сохранение объекта в `atomic` —
  один POST после commit, rollback — ни одного.
- A01–A21, C-подмножество, H01–H06, S01–S08 (`django`-стор) зелёные на всей матрице §5; mypy strict с django-stubs; coverage-floor.
- README EN/RU + docs §6; `ReadmeAssertions`; запись в `docs/spec/README.md` и семейной таблице «Other packages» всех README.
