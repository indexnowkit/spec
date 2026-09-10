# 22. Python: `indexnowkit-sqlalchemy` (волна P, шаг 3)

Статус: **переписана 2026-09-10** (прежняя редакция 2026-09-03 заменена). Базируется на спеке 20; эталон — `indexnowkit/doctrine`
(unit of work + staging, спека 11) и `Transaction\TransactionStaging` core PHP. Решения — §9.

## 0. Цель и границы

`install(Session | sessionmaker | async_sessionmaker, kit)` — три слушателя на классе сессии, и объекты с `@indexnow`-правилами
уходят после **реального** commit, savepoint-откаты учтены, `deleted` резолвится до удаления. Без фреймворка (SQLAlchemy 2.0/2.1,
sync и async). Роутера у SQLAlchemy нет: правила `route` работают только с `RouteUrlResolver`, который подставит FastAPI/Flask
(23/24); без него — `url`/`urls`/`resolver` и `base_url`. Границы: Flask-SQLAlchemy `models_committed` не используется (см. §8);
SQLModel — тот же `Session` (ничего специального); Core-level `insert()/update()` без ORM — A13.

## 1. Факты (проверены 2026-09-10, docs.sqlalchemy.org/en/20 и /en/21)

1. **Версии**: 2.0.52 (2026-08-11, ≥3.7); **2.1.0rc2** 2026-09-08 (rc1 2026-08-31), `requires_python >=3.11`, `greenlet` только в extra
   `[asyncio]`; 2.1 — `execution_options` у `Session`/`sessionmaker`/`AsyncSession`, `RegistryEvents`, ORM-строки как кортежи
   (типизация `Select[int, str]`), без переименований событий сессии и без изменений `session.info`
   (docs.sqlalchemy.org/en/21/changelog/migration_21). `aiosqlite` 0.22.1 (2025-12-23).
2. **События сессии** (orm/session_events, orm/events `SessionEvents`): `before_flush(session, flush_context, instances)` — можно
   менять состояние; `session.new/dirty/deleted` — до записи; `after_flush` — SQL уже ушёл, коллекции ещё pre-flush;
   `after_flush_postexec` — коллекции пусты; `after_commit(session)` — **реальный** DBAPI-commit, не RELEASE SAVEPOINT; сессия вне
   транзакции, **SQL из обработчика нельзя**; `after_rollback` — реальный DBAPI-rollback; `after_soft_rollback` — любой `rollback()`,
   включая без DBAPI; `after_transaction_create/end(session, transaction)` — для каждого `SessionTransaction`, **включая nested**
   (`transaction.nested`, `transaction.parent`); `after_begin(session, transaction, connection)`.
3. **`begin_nested()`** (orm/session_transaction «Using SAVEPOINT»): SAVEPOINT в текущей транзакции, объект `SessionTransaction`;
   `.commit()` = RELEASE, `.rollback()` = ROLLBACK TO; перед SAVEPOINT сессия **безусловно flush'ит** pending-состояние; внешняя
   транзакция продолжается. Тесты: «join into external transaction» — `Session(bind=connection, join_transaction_mode="create_savepoint")`.
4. **История атрибутов**: `sqlalchemy.inspect(obj).attrs[name].history` → `has_changes()`, `.added`, `.deleted`, `.unchanged`;
   `session.is_modified(obj)`; `session.dirty` включает объекты без реальных изменений (документировано) — фильтр через history.
5. **Async** (orm/extensions/asyncio): `AsyncSession` оборачивает sync `Session`; события регистрируются на sync-классе
   (`Session` глобально) или через `async_sessionmaker(sync_session_class=…)` / `AsyncSession.sync_session_class`; обработчики
   выполняются **синхронно в greenlet-контексте**, IO адаптируется прозрачно (но `after_commit` SQL всё равно нельзя — п. 2);
   рекомендация `expire_on_commit=False` для `AsyncSession` (иначе доступ к атрибуту после commit — implicit IO → ошибка);
   `session.run_sync()`.
6. **`session.info`** — словарь на сессию для пользовательского состояния (документирован в `Session` API); изоляция на сессию —
   потокобезопасность обеспечивает сама сессия (не shared между потоками по контракту SQLAlchemy).

## 2. Принципы

1. **Резолв в `before_flush`/`after_flush`, доставка в `after_commit`, только примитивы между ними.** После commit атрибуты
   могут быть expired (implicit IO для async) — в `after_commit` нет обращений к объектам.
2. **Staging учитывает savepoint'ы.** Не «очистить всё при soft rollback», как в старой редакции: `after_transaction_create/end`
   дают точный стек транзакций — URL привязываются к кадру, откат кадра выбрасывает только его (A05c: внешние URL остаются).
   Это `Transaction\TransactionStaging` core PHP с `StagingFrame` (спека 11), переносится как `SessionStaging`.
3. **Идемпотентная установка на классе, не на экземпляре.** `install()` дважды — no-op; на `sessionmaker` — на его `class_`.
4. **Ничего не бросать из событий.** `ObserverHelper.guard`.
5. **Ноль зависимостей кроме `indexnowkit` и `sqlalchemy>=2.0`.**

## 3. Дизайн

### 3.1. Пакет

`python/packages/indexnowkit-sqlalchemy/`, модуль `indexnowkit_sqlalchemy`, `dependencies = ["indexnowkit>=0.1,<0.2", "sqlalchemy>=2.0"]`,
`requires-python >=3.11`. Классы: `install()`/`uninstall()`, `Listener` (три+два события), `SessionStaging` (кадры по
`SessionTransaction`), `SqlAlchemySubjectReader` (`inspect(obj).attrs`, `.history` для старого значения — старое состояние
**без запроса**), `IndexNowModelMixin` (маркер + `get_indexnow_urls()` по умолчанию для правила `url`), `submit_query(session, select_stmt,
event)` (A13, чанки через `session.execute(stmt).scalars().partitions()`? — `yield_per`), `Check`-строки.

### 3.2. Объявление модели

```python
@indexnow_defaults(when="published", fields=["slug", "title", "published"])
@indexnow(url="public_url")                       # метод/свойство → str | Iterable[str] | None; относительный → base_url
class Post(IndexNowModelMixin, Base):
    __tablename__ = "posts"
    def public_url(self) -> str: return f"/posts/{self.slug}"

registry.register(Legacy, rules=[indexnow(url=lambda o: f"/legacy/{o.id}")])
```

Правила `route` — только с роутером адаптера 23/24 (`install(..., router=...)`); без него `check` предупреждает (`router.missing`)
и правило даёт пустой список с `error` в логе (как PHP без `RouteUrlResolver`). `via` — по relationship (`post.category`,
коллекции `post.tags`); ленивая загрузка в `before_flush` — обычный SQL внутри транзакции (разрешён), `max_via_fanout` ограничивает.

### 3.3. Хуки и commit-safety

- `before_flush(session, ctx, instances)`: обход `session.new`, `session.dirty` (фильтр `is_modified` + history по полям правил),
  `session.deleted`; `changes.created_events`/`updated_events(obj, changed, change_set)`/`deleted(obj)`; **резолв здесь** (объект жив,
  старые значения — в `history.deleted`, id для новых объектов ещё нет → отложить `created` на `after_flush`);
  `renamed()` для изменённых параметров маршрута; результат — `list[str]` в `staging.stage(session, urls)`.
- `after_flush(session, ctx)`: резолв отложенных `created` (id есть, коллекции ещё pre-flush — объекты доступны); `stage()`.
- `after_transaction_create(session, tx)`: `staging.open_frame(session, tx)` (для `tx.nested` — дочерний кадр); `after_transaction_end(session, tx)`:
  `tx.nested` → кадр закрывается: если откат (`session.is_active`? нет — по факту `after_soft_rollback` между) — URL кадра
  выброшены, иначе слиты в родителя; внешний `tx` — кадр закрыт, URL ждут `after_commit`/`after_rollback`.
- `after_commit(session)`: `urls = staging.take(session)` → `kit.collect(urls)` (в scope запроса — flush адаптером 23/24; без scope —
  сразу диспетчер). Никаких обращений к объектам.
- `after_rollback(session)` и `after_soft_rollback(session, previous_transaction)`: `staging.discard(session, previous_transaction)`
  — только кадр отката.
- Состояние — `session.info["indexnowkit"]` (`SessionStaging` кладёт свой объект; `session.info` документирован для этого).
- Autocommit-режим (`session.commit()` без явного `begin`): SQLAlchemy 2.0 всегда в транзакции «begin once» — `after_commit` приходит.

Итог: A01–A06, A05b (один POST — все кадры слиты в корневой), A05c (внутренний откат — только внешние URL) — без документированных
упрощений старой редакции.

### 3.4. Async

`install(async_sessionmaker)` → регистрация на `sync_session_class` (факт §1.5); обработчики синхронные, `before_flush` может
трогать relationship (greenlet адаптирует IO); `after_commit` — ничего не читает. `kit.collect()` — синхронный вызов; диспетчер
`asyncio` (`AsyncTaskDispatcher`) — `loop.create_task(kit.asubmit)` внутри greenlet: `asyncio.get_running_loop()` доступен
(обработчик выполняется в потоке loop) — проверить (§7.2); иначе `thread`/`callable`. README: `expire_on_commit=False`.

### 3.5. Bulk (A13)

`session.execute(update(Post)…)`, `bulk_insert_mappings`, Core `insert()` — не проходят unit of work. `submit_query(session, select(Post)
.where(...), event="updated", batch=1000)` + `kit.submit_objects`.

### 3.6. Диагностика

`Check`-строки: `wiring.sqlalchemy` (слушатели установлены на классе X, моделей с правилами N), `router.missing` (правила `route` без
роутера), `sqlalchemy.version`. Команд у пакета нет — `python -m indexnowkit` / CLI адаптера 23/24; `explain(obj)` — метод core.

## 4. BC и версии

`indexnowkit-sqlalchemy` 0.1.0; `sqlalchemy>=2.0,<2.2` (2.1 rc в матрице как `experimental`); публичное — `install`, `uninstall`,
`IndexNowModelMixin`, `submit_query`, `SqlAlchemySubjectReader`; `SessionStaging` — internal.

## 5. Тесты

pytest; sqlite in-memory sync и `aiosqlite` (`pytest-asyncio` 1.4 `asyncio_mode = "auto"`); фикстуры кита: post, multi-post (getter
`is_published` над `published`), categorized post (`category` relationship, `tags` M2M через `secondary`), category, tag, untracked,
broken, bad-decorator; **`OrmConformance`-драйвер**: `begin()` = `session.begin()` / `begin_nested()` для вложенного, `commit()`/`rollback()`
— соответствующего `SessionTransaction`, `flush()` = `kit.flush()`, `bulk_update_title()` = `session.execute(update(...))`; A01–A21,
A05b/A05c, A10b; `test_async.py` — тот же драйвер над `AsyncSession` через `run_sync`? Нет — драйвер async, тесты `await`;
`test_install.py` (идемпотентность, sessionmaker, `uninstall`); `test_expire_on_commit.py` (True — ничего не читаем после commit,
тест на отсутствие `DetachedInstanceError`); PostgreSQL job (savepoint-семантика на реальной БД: A05c). Матрица: SQLAlchemy 2.0 ×
(3.11–3.14), 2.1rc × (3.11, 3.14) experimental.

## 6. Документация

README EN/RU (install 3 строки, модель, «After commit, savepoints included», async, bulk, FastAPI/Flask ссылки, AI-notes);
`docs/`: `events.md` (какие события и почему `after_commit` не читает объекты), `async.md`, `models.md`, `testing.md` (join into
external transaction + `FakeTransport`), `troubleshooting.md`, `bc.md`.

## 7. Риски и что проверить первым

1. **`after_transaction_end` для nested при откате** — порядок событий: `after_soft_rollback` → `after_transaction_end(nested)`?
   Проверить на 2.0.52 и 2.1rc2 unit-тестом до дизайна `SessionStaging`; если порядок обратный — кадр помечается «откачен» в
   `after_soft_rollback(previous_transaction)` и закрывается в `after_transaction_end`.
2. **`asyncio.get_running_loop()` из greenlet-обработчика** — да/нет решает, работает ли `AsyncTaskDispatcher` из `after_commit`.
3. **`before_flush` при `begin_nested()`** — безусловный flush (факт §1.3) вызывает `before_flush` до SAVEPOINT: URL попадают во
   внешний кадр (правильно: запись — во внешней транзакции до savepoint).
4. **`session.dirty` без изменений** — фильтр `is_modified`/history обязателен (A12).
5. **Многие сессии в потоках** (`scoped_session`) — staging в `session.info` каждой; ok.
6. **2.1 и типизация** — `Select[…]` не влияет; `mypy --strict` с плагином? SQLAlchemy 2.x типизирован без плагина.

## 8. Не делать (рассмотрено)

- `models_committed` Flask-SQLAlchemy (требует `SQLALCHEMY_TRACK_MODIFICATIONS`, пакет без релизов с 2023) — события SQLAlchemy.
- Своя таблица истории/дебаунса в БД пользователя — `sqlite`-файл core или стор адаптера 23/24; по спросу — `SqlAlchemySubmissionStore`
  (таблица через `metadata.create_all`) отдельной волной.
- Поддержка SQLAlchemy 1.4 — 2.0 вышел 2023-01, 1.4 EOL.

## 9. `[решение]` — рекомендации

1. **Staging по кадрам транзакций (`after_transaction_create/end`), не «очистить при soft rollback»** — да; альтернатива старой
   редакции теряет внешние URL при внутреннем откате (A05c красный или «документированное упрощение»).
2. **Старое состояние из `attrs.history`, без запроса** — да.
3. **Правила `route` только с роутером адаптера 23/24; в чистом SQLAlchemy — `url`/`resolver` + `base_url`** — да.
4. **Порядок в волне: после Django** (3-й) — да (спека 26 §3).

## 10. Definition of Done

- `install(Session, kit)` в скрипте на sqlite: create/commit → POST; `begin_nested` + откат → внешние URL ушли, внутренние нет;
  async на `aiosqlite` — те же сценарии.
- A01–A21 (+b/c) sync и async зелёные; PostgreSQL job; mypy strict; README EN/RU + docs; coverage-floor.
