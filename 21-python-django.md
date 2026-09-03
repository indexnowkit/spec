# 21. Python: `indexnowkit-django`

Django ≥ 4.2 (LTS), 5.1, 5.2, 6.0. Python ≥ 3.11. Зависимость: `indexnowkit`, `django`.

## Установка

```python
INSTALLED_APPS += ["indexnowkit_django"]
INDEXNOW = {
    "KEY": env("INDEXNOW_KEY"),
    "BASE_URL": "https://www.example.com",   # или SITE_ID + django.contrib.sites
    "DISPATCH": "sync",                       # sync | thread | callable path "myapp.tasks.indexnow_submit"
    "DEBOUNCE_STORE": "cache",                # cache | memory
}
urlpatterns += [path("", include("indexnowkit_django.urls"))]   # GET /<key>.txt
```

Ключи settings = ключи единой схемы из 02 в UPPER_SNAKE.

## Объявление модели

```python
from indexnowkit_django import IndexNowMixin, indexnow

class Post(IndexNowMixin, models.Model):
    def get_absolute_url(self): return reverse("post", args=[self.slug])
    indexnow_when = lambda self: self.status == "published"
    indexnow_fields = {"title", "body", "slug", "status"}

@indexnow.register(url=lambda a: a.get_absolute_url(), when=...)   # альтернатива без миксина
class Article(models.Model): ...
```

Также `indexnow.register(Model, ...)` вызовом в `AppConfig.ready()` для чужих моделей.

## Хуки и commit-safety

- `AppConfig.ready()` подключает `post_save`, `post_delete`, `pre_delete`, `pre_save` только
  к зарегистрированным моделям (`sender=Model`), не глобально.
- `pre_save` при `on_fields`: загрузить старые значения `Model.objects.filter(pk=...).values(*fields)`
  один запрос; сравнить в `post_save`. Опционально `django-model-utils FieldTracker`, если
  установлен, использовать его.
- `pre_delete`: вычислить URL, сложить в `instance._indexnow_urls`. `post_delete`: взять оттуда.
- Публикация: переход `when: True → False` детектится через тот же снимок старых полей
  (если `status` входит в `indexnow_fields`), трактуется как `deleted` (URL шлётся).
- Отправка: `transaction.on_commit(lambda: collector.add(urls), using=db_alias)`.
  Семантика Django: колбэк выполняется после **внешнего** `atomic`, откат savepoint
  отбрасывает колбэк. Вне транзакции (autocommit) выполняется сразу. Это даёт A01–A06.
- Flush коллектора: сигнал `request_finished` (в `on_commit` мы только кладём в коллектор;
  если on_commit сработал вне запроса, например в management command или Celery-задаче,
  `Collector.add` при отсутствии активного scope сразу вызывает dispatcher).
- `DISPATCH=sync`: HTTP в `request_finished` после отправки ответа (WSGI-серверы уже отдали
  ответ, но воркер занят на время POST; timeout по умолчанию 10 s, рекомендация в README
  ставить 3 s для sync). `thread`: daemon thread. `callable`: dotted path на функцию
  `fn(urls: list[str]) -> None`, в README пример с `@shared_task` Celery и `django-q2`.
- Ограничение (A13): `QuerySet.update`, `bulk_create`, `bulk_update`, raw SQL сигналы не
  вызывают. В README: `indexnow.submit_queryset(qs)`.

## Serving ключа

`indexnowkit_django.urls`: `re_path(r"^(?P<key>[A-Za-z0-9-]{8,128})\.txt$", key_file)`, view
сравнивает `key` с ключами из конфига (включая `HOSTS`), иначе 404. `text/plain; charset=utf-8`,
`Cache-Control: public, max-age=86400`.

## Management commands

- `indexnow_key`: печатает новый ключ, флаг `--write .env` (append `INDEXNOW_KEY=`).
- `indexnow_check`: валидирует settings, GET ключа по `BASE_URL`, тестовый dry-run POST.
- `indexnow_submit <url>...`: ручная отправка.
- `indexnow_sitemap [--sitemap-url|--from-django-sitemaps]`: второй режим обходит
  `django.contrib.sitemaps` из `urlpatterns` (импортирует sitemaps dict) и шлёт все `location()`.
  Флаг `--changed-since 1d` по `lastmod`.

## Checks framework

`indexnowkit_django.checks` с тегом `indexnow`: `indexnow.E001` нет KEY при ENABLED,
`E002` формат ключа, `E003` BASE_URL не абсолютный, `W001` DISPATCH=sync в production
(DEBUG=False) с рекомендацией callable, `W002` модель зарегистрирована без `get_absolute_url`
и без `url=`.

## Интеграция с sitemaps

Опционально: `IndexNowMixin` не трогает sitemap. В README рецепт: `Sitemap.lastmod` из
`updated_at`, чтобы Google получал сигнал из sitemap.

## DRF

Ничего специального: `serializer.save()` вызывает `Model.save()` → сигналы. `ATOMIC_REQUESTS`
совместим с `on_commit`.

## Тесты

pytest-django, матрица tox: py311–314 × Django 4.2/5.1/5.2/6.0. Conformance A01–A14, H01–H06.
Тест A05 через вложенный `atomic()` с `savepoint=True` и `set_rollback`.

## Конкуренты

На PyPI пакета `django-indexnow` нет. `wagtail-indexnow` только Wagtail.
