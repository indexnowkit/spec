# 25. Python: `indexnowkit-wagtail` (волна P, шаг 6, по решению)

Статус: **переписана 2026-09-10** (прежняя редакция 2026-09-03 — «PR в wagtail-indexnow, иначе пакет через 30 дней» — заменена
после проверки апстрима). Тонкая обёртка над спекой 21. Решения — §9.

## 0. Цель и границы

Wagtail публикует страницы через ревизии: `page.save()` — черновик, `revision.publish()` — публикация; сигналы `post_save` не
означают «страница видна», поэтому Django-адаптер (сигналы модели) для Wagtail недостаточен: нужны сигналы публикации. Пакет
подключает `page_published`/`page_unpublished`/`page_slug_changed`/`post_page_move` (и `published`/`unpublished` для сниппетов) к
`ObjectChangeHandler` core, URL — `page.full_url` (Wagtail `Site` даёт host), мультисайт — `hosts`. Всё остальное (settings, key-view,
команды, checks, dispatch, история) — `indexnowkit-django`. Границы: не форк `wagtail-indexnow`; без UI в админке Wagtail (по спросу).

## 1. Факты (проверены 2026-09-10)

1. **Wagtail 8.0** (2026-08-25, ≥3.10, `Django>=5.2`; PyPI). Классификаторы `Framework :: Wagtail :: 8` есть.
2. **Сигналы** (docs.wagtail.org/en/stable/reference/signals): `page_published(sender=PageClass, instance, revision)`,
   `page_unpublished(sender, instance)`, `published`/`unpublished` — для любой ревизионной модели (сниппеты), `page_slug_changed(sender,
   instance, instance_before)` — при публикации смены slug, `post_page_move(sender, instance, url_path_before, url_path_after,
   parent_page_before, parent_page_after)`. Все — Django-сигналы, синхронные.
3. **`wagtail-indexnow` 0.2.0** (RealOrangeOne = Jake Howard, 2025-08-06; 1★, 2 релиза за 2 года, PyPI 1 111 загрузок/мес; BSD-3;
   `wagtail>=6.3`, `Django>=4.2`, `requests`): хуки `before_publish_page`/`after_publish_page` (не сигналы), условие — `last_published_at`
   старше 10 минут, ключ = `"indexnow-" + pbkdf2(__name__, INDEXNOW_KEY или SECRET_KEY)` (ротация `SECRET_KEY` = смена ключа), один
   POST на `api.indexnow.org` с одним URL, `raise_for_status()` **внутри хука** (ошибка IndexNow роняет публикацию), нет unpublish/
   delete/move/slug, нет батчей, 202/429, `check`, мультисайта ключей (host из URL, ключ один). Последний коммит 2025-08-06
   («Version 0.2.0», PR #1 от стороннего автора). Не архивирован.

## 2. Принципы

Тонкость (≤ 250 строк); всё через `indexnowkit-django` (зависимость, не дублирование); publish-семантика Wagtail, а не `post_save`;
never-throw в хуке публикации (в отличие от апстрима); честная ссылка на `wagtail-indexnow` в README («если вам нужен один URL на
публикацию без зависимостей — он»).

## 3. Дизайн

- `python/packages/indexnowkit-wagtail/`, модуль `indexnowkit_wagtail`, `dependencies = ["indexnowkit-django>=0.1,<0.2", "wagtail>=6.3"]`
  (6.3 LTS? Wagtail LTS — 6.3 до 2026-02? проверить на docs.wagtail.org/en/stable/releases/upgrading — §7.1; минимум решает матрица),
  `requires-python >=3.11`.
- `AppConfig.ready()`: подключает пять сигналов **глобально** (все `Page` публикуемы), `registry.register_factory()` core — для
  экземпляра `Page` возвращает правило `url` = `full_url` (event по сигналу: `page_published` → `created`, если `revision`
  первая / иначе `updated`; `page_unpublished` → `deleted`; `page_slug_changed` → `renamed(instance, {slug: (before, after)})`:
  старый `full_url` из `instance_before` как `deleted`; `post_page_move` → старый `url_path_before`+site → `deleted`, новый — `updated`;
  дочерние страницы при move — их URL тоже меняются: обход `instance.get_descendants()` с лимитом `resolver.max_via_fanout`,
  документировать).
- `INDEXNOW["wagtail"] = {"exclude": ["app.ModelName"], "snippets": True}` — исключения и сниппеты (`published`/`unpublished` для
  моделей с `RevisionMixin`+`get_url`? у сниппетов нет URL по умолчанию — только если модель несёт `@indexnow`-правило; без него
  ничего).
- Commit-safety: `page_published` шлётся из `revision.publish()` внутри `transaction.atomic()` Wagtail — `on_commit` (через
  `deliver` Django-адаптера) обязателен, как в спеке 21.
- Мультисайт: `Site.find_for_request`/`page.get_site()` → host; ключ — `hosts` карта core; `check` строка `wagtail.sites` (сайты без
  ключа при `strict_hosts` — warning).
- Команды/`check`/history — из `indexnowkit-django`; добавляется `indexnow_submit_pages [--site --live-only]` (все опубликованные
  страницы дерева — «bulk после миграции»).

## 4. BC и версии

`indexnowkit-wagtail` 0.1.0; `wagtail>=6.3,<9`; матрица Wagtail 6.3 (если поддерживается апстримом) / 7.x / 8.0 × Django 5.2/6.x.

## 5. Тесты

pytest-django + `wagtail.test.utils`: публикация (A01-аналог), unpublish → `deleted`, slug change → старый+новый, move, черновик
(`save_revision()` без publish) → ничего, исключённая модель, мультисайт с двумя `Site` и `hosts`; H01–H06 через `indexnowkit-django`
(не дублируются, но `test_ids` требует H у адаптера — ставится помеченный `H0x: inherited from indexnowkit-django` пустой тест? Нет —
запуск ассерций django-адаптера над `wagtail`-проектом: 6 тестов).

## 6. Документация

README EN/RU (install, `INSTALLED_APPS`, `INDEXNOW`, «Why not wagtail-indexnow» — таблица различий из §1.3 честно, AI-notes), `docs/`:
`signals.md`, `multi-site.md`, `bc.md`. Регистрация в каталоге пакетов Wagtail (wagtail.org/packages) и djangopackages.

## 7. Риски

1. **Матрица Wagtail**: какие версии в поддержке (LTS-политика Wagtail) — прочитать до реализации; минимум может быть 7.0.
2. **`page_published` для первой публикации vs обновления** — `revision`-объект: первая ли? (`instance.first_published_at == last_published_at`?)
   — для IndexNow разницы нет (оба = отправка); `created`/`updated` — только для `explain`.
3. **Move поддерева** — сотни URL: fanout-лимит + предупреждение + `indexnow_submit_pages` для остального.
4. **Двойная отправка** с `indexnowkit-django` сигналами `post_save` на `Page`: правила на `Page` через реестр-фабрику — `post_save`
   адаптера Django для `Page` **отключается** пакетом (страница без публикации не публична): регистрация исключения в реестре.

## 8. Не делать

PR в `wagtail-indexnow` (1★, минимализм — не примут замену HTTP-слоя; спека 91 «открытое решение 2» закрывается); форк; хуки
`before/after_publish_page` вместо сигналов (хуки — про UI-запрос, сигналы — про модель, срабатывают и из кода/команд).

## 9. `[решение]` — рекомендации

1. **Свой пакет вместо PR в `wagtail-indexnow`** — да (факты §1.3: апстрим минимален по замыслу, один мейнтейнер, `raise_for_status` в
   хуке); альтернатива PR — 30-дневное ожидание с малой вероятностью принятия и без дебаунса/батчей/`check` в итоге.
2. **Ставится последним (6-й), опционально в волне** — да; можно вынести в волну P2 без ущерба остальному.

## 10. Definition of Done

Wagtail-проект из README: публикация страницы → POST после commit; unpublish → POST (deleted); slug → два URL; тесты §5; README;
регистрация в каталогах.
