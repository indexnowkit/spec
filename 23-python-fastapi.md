# 23. Python: `indexnowkit-fastapi` (волна P, шаг 4)

Статус: **переписана 2026-09-10** (прежняя редакция 2026-09-03 заменена). Тонкая обёртка над спеками 20 и 22. Решения — §9.

## 0. Цель и границы

Что пакет добавляет поверх `indexnowkit` + `indexnowkit-sqlalchemy` и ради чего существует (иначе — рецепт в README core):

1. `IndexNowKit` как зависимость FastAPI (`Depends(get_indexnow)`) и объект в `app.state`, созданный в `lifespan` вместе с
   `httpx.AsyncClient`, закрытый на shutdown.
2. Маршрут `/{key}.txt` через `asgi.KeyFileApp` core одной строкой (`include_in_schema=False`).
3. Scope коллектора на запрос и flush **после** ответа — `asgi.CollectorMiddleware` core, подключённый `app.add_middleware`.
4. `RouteUrlResolver` над `app.url_path_for(name, **params)` + `base_url` → правила `route` в SQLAlchemy-моделях работают.
5. `install_sqlalchemy(session_factory)` — делегат в спеку 22 с роутером п. 4 и `AsyncTaskDispatcher`/`callable` по конфигу.
6. Конфигурация из `pydantic-settings`? — нет (§8): `Config.from_env()` core + `from_mapping(settings.model_dump())` в README для тех,
   у кого настройки в pydantic.

Границы: без своего CLI (`python -m indexnowkit`); без Starlette-only API (пакет для FastAPI, но всё ASGI-общее живёт в core и
работает с голым Starlette/Litestar по рецепту).

## 1. Факты (проверены 2026-09-10)

1. FastAPI 0.141.1 (2026-07-29, ≥3.10, `starlette>=0.46`, `pydantic>=2.9`); Starlette 1.6.0 (2026-08-08) — release notes не
   прочитаны (starlette.io не резолвился), §7.1.
2. **Lifespan** (fastapi.tiangolo.com/advanced/events): `FastAPI(lifespan=asynccontextmanager)`, код до `yield` — старт, после —
   остановка; `on_event("startup"/"shutdown")` deprecated; при `lifespan` `on_event` не вызываются; состояние — `app.state` или yield-словарь.
3. **BackgroundTasks** (tutorial/background-tasks): выполняются после отправки ответа **в том же процессе**; для тяжёлого — Celery;
   о потере при падении процесса документация молчит (в памяти — теряются). Не батчится между запросами → core `asyncio`-диспетчер
   или `callable`, не `BackgroundTasks` (как в старой редакции).
4. **ASGI**: `http.response.body` с `more_body: False` — конец ответа; middleware после `await app(scope, receive, send)` работает
   после отправки (asgi.readthedocs.io); `asyncio.create_task` требует строгой ссылки (спека 20 §1.2).
5. Тесты: `httpx.AsyncClient(transport=ASGITransport(app))`, `pytest-asyncio` 1.4.0.

## 2. Принципы

Тонкость: ≤ 300 строк; всё общее — в core `asgi`; ни одного собственного текста `check`; конфигурация — `Config` core; FastAPI-версии
— `>=0.110`; async-first, sync-сессии SQLAlchemy тоже поддерживаются.

## 3. Дизайн

```python
from indexnowkit_fastapi import IndexNowKitPlugin
plugin = IndexNowKitPlugin(Config.from_env())           # или Config(...)
app = FastAPI(lifespan=plugin.lifespan)                  # создаёт AsyncHttpxTransport (если httpx) и kit; закрывает на shutdown
plugin.install(app)                                      # /{key}.txt, CollectorMiddleware, app.state.indexnow, роутер url_path_for
plugin.install_sqlalchemy(SessionLocal)                  # спека 22 с роутером и диспетчером по dispatch

@app.post("/posts")
async def create(indexnow: IndexNowKit = Depends(plugin.dependency)): ...
```

- `plugin.lifespan` композируется с пользовательским (`contextlib.AsyncExitStack`; README: «свой lifespan — вызовите
  `async with plugin.lifespan(app):`»); `await plugin.startup(app)` / `shutdown(app)` — ручной путь.
- `Services` слоя 2: `transport` → `AsyncHttpxTransport` при `httpx`, иначе `UrllibTransport` + `to_thread`; `router` →
  `FastApiRouteUrlResolver(app)` (`url_path_for`, `RouteOrigin.root`); `dispatch`: **`sync` (дефолт адаптера)** — `asgi.CollectorMiddleware`
  делает `await kit.asubmit(urls)` **после** `http.response.body` с `more_body: False`: ответ у клиента, loop не заблокирован
  (httpx async; без httpx — `to_thread`), task запроса живёт до конца отправки, и graceful shutdown uvicorn его ждёт; `asyncio` —
  opt-in (отдельная задача на loop, теряется при остановке — `check` предупреждает `dispatch.asyncio`), `callable`, `none`; `debounce.store`: `memory` (дефолт),
  `sqlite` (путь), объект с `get/set/add` из `plugin = IndexNowKitPlugin(config, cache=redis_like)`.
- `check`: `python -m indexnowkit check` с `INDEXNOW_*` — но роутер и wiring живут в приложении → `plugin.check_command(app)`
  печатает то же через `CheckRunner` (регистрируется как `python -m myapp.indexnow check`? — README-рецепт из 5 строк на
  `typer`/`argparse` пользователя; своего CLI у пакета нет). Строки адаптера: `wiring.asgi` (middleware подключён), `router.fastapi`.
- Ключ-файл: `app.add_route("/{key}.txt", KeyFileApp(...))` — pattern `{key}` Starlette не ограничивает по регулярке, проверка —
  в `KeyFileResponder` (404 иначе); `include_in_schema=False`.

## 4. BC и версии

`indexnowkit-fastapi` 0.1.0; `dependencies = ["indexnowkit>=0.1,<0.2", "fastapi>=0.110"]`, `optional-dependencies.sqlalchemy = ["indexnowkit-sqlalchemy>=0.1,<0.2"]`,
`[httpx]` → `indexnowkit[httpx]`; `requires-python >=3.11`. Публичное — `IndexNowKitPlugin` и его методы.

## 5. Тесты

H01–H06 (`test_http.py` через `ASGITransport`; H06 — порядок: тело ответа получено клиентом **до** POST в `FakeTransport` — фиксируется
временем/счётчиком в middleware-тесте), A01–A21 через драйвер спеки 22 с роутером (правила `route`), `test_lifespan.py` (создание/закрытие
транспорта, композиция), `test_dispatch_sync.py` (POST после ответа, loop свободен: параллельный запрос обслуживается во время `asubmit`; shutdown ждёт хвост),
`test_dispatch_asyncio.py` (задача выполнена, ссылка удержана, shutdown с pending — warning), `test_readme.py`.
Матрица: FastAPI latest × (3.11–3.14) + `lowest` 0.110 на 3.11.

## 6. Документация

README EN/RU (10 строк установки, модель SQLAlchemy с `route`, `check`, AI-notes), `docs/`: `lifespan.md`, `dispatch.md`
(`asyncio` vs Celery/arq/taskiq через `callable`), `starlette.md` (голый Starlette/Litestar на core `asgi`), `testing.md`, `bc.md`.

## 7. Риски

1. **Starlette 1.x** — прочитать release notes до реализации (изменения middleware/`Request.state`?); `add_middleware` порядок —
   `CollectorMiddleware` внешним.
2. **`url_path_for` без host** — `RouteOrigin.root(config, host)` обязателен: `base_url` или `hosts.<host>.base_url` → `check`
   `config.base_url` error при правилах `route`.
3. **Sync-эндпоинты в threadpool** — `contextvars` копируются Starlette в `run_in_threadpool`; scope виден. Проверить.

## 8. Не делать

`BackgroundTasks` как диспетчер (не батчится, теряется); `pydantic-settings` в зависимостях; свой CLI; поддержка `on_event`.

## 9. `[решение]` — рекомендации

1. **Пакет нужен** (шесть пунктов §0 — ~250 строк, но именно они делают «fastapi indexnow» пятиминутной установкой и дают
   PyPI/SEO-имя) — да; альтернатива «только рецепт в core» — не ранжируется по запросу и повторяется у каждого пользователя.
2. **Дефолт `dispatch: sync` как `await asubmit()` после ответа** — принято 2026-09-10 (адверсальный проход, спека 26 §9.11); было
   `asyncio`: detached-задача режется при остановке сервера, а awaited-хвост запроса не блокирует loop и переживает graceful shutdown.
3. **Ставится 4-м, до Flask** — да.

## 10. Definition of Done

`uvicorn`-приложение из README: `GET /<key>.txt` 200; `POST /posts` → один POST в мок после ответа; H01–H06, A01–A21 зелёные;
mypy strict; README EN/RU; docs.
