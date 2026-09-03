# 22. Python: `indexnowkit-sqlalchemy`

SQLAlchemy ≥ 2.0, sync и async. Зависимость: `indexnowkit`, `sqlalchemy>=2.0`.

## Установка

```python
from indexnowkit_sqlalchemy import install, IndexNowMixin
install(Session, indexnow=IndexNow(Config.from_env()))       # класс или sessionmaker или engine-bound
# async: install(async_session_maker)  # регистрирует на sync_session_class
```

`install` регистрирует три слушателя на классе сессии (не на экземплярах), идемпотентно.

## Объявление модели

```python
class Post(IndexNowMixin, Base):
    __tablename__ = "posts"
    def get_indexnow_urls(self) -> list[str]: return [f"/posts/{self.slug}"]
    indexnow_when = lambda self: self.published
    indexnow_fields = {"slug", "title", "body", "published"}

registry.register(Legacy, url=lambda o: f"/legacy/{o.id}")   # без миксина
```

URL относительный, `base_url` из Config (SQLAlchemy не знает роутер; FastAPI/Flask адаптеры
могут подставить `url_path_for`/`url_for` в резолвере, см. 23/24).

## Хуки и commit-safety

Три события `SessionEvents`:

1. `before_flush(session, ctx, instances)`: обход `session.new | session.dirty | session.deleted`,
   фильтр по registry. Для `dirty` проверить реальные изменения через
   `sqlalchemy.inspect(obj).attrs[f].history.has_changes()` по `indexnow_fields`
   (иначе `session.dirty` содержит и объекты без изменений). Вычислить URL **здесь**, положить
   примитивы в `session.info["indexnowkit"]: set[str]`. Для `deleted` то же (объект ещё жив).
2. `after_commit(session)`: `urls = session.info.pop("indexnowkit", None)`; передать в
   `Collector.add` или напрямую в dispatcher. Причина: `expire_on_commit=True` делает
   атрибуты недоступными после commit, для `AsyncSession` это ещё и implicit IO. Поэтому
   в `after_commit` только примитивы.
3. `after_rollback(session)`: очистить `session.info["indexnowkit"]`. Для savepoint-отката
   (`after_soft_rollback`) очистка тоже, документированное упрощение: URL, собранные во
   внешней части транзакции до savepoint, теряются. Точное поведение A05 тестируется как
   «POST нет»; частичная потеря в A06-подобном сценарии с savepoint документирована.

`session.info` изолирован на сессию, потокобезопасность обеспечивается самим SQLAlchemy.

## Async

`install(AsyncSession)` регистрирует на `AsyncSession.sync_session_class` (или переданный
`sessionmaker(class_=AsyncSession).kw["sync_session_class"]`). Dispatcher по умолчанию
`ThreadDispatcher`, чтобы не блокировать event loop; при установленном httpx можно
`AsyncDispatcher` (создаёт task через `asyncio.get_running_loop().create_task`, если есть loop).

## Bulk

`session.execute(update(Post)...)`, `bulk_insert_mappings` не проходят через unit of work,
события не срабатывают (A13). README: `indexnow.submit([...])`.

## Тесты

pytest, sqlite in-memory (sync и aiosqlite), матрица SQLAlchemy 2.0.x/2.1. Conformance A01–A14.
