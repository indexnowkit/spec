# 23. Python: `indexnowkit-fastapi`

FastAPI ≥ 0.110, Starlette. Зависимости: `indexnowkit[httpx]`, `indexnowkit-sqlalchemy`
(optional extra `[sqlalchemy]`).

## Установка

```python
from indexnowkit_fastapi import IndexNowPlugin
plugin = IndexNowPlugin(Config.from_env())
app = FastAPI(lifespan=plugin.lifespan)        # создаёт/закрывает httpx.AsyncClient
plugin.install(app)                            # роут /{key}.txt, middleware-collector
plugin.install_sqlalchemy(SessionLocal)        # делегирует в 22, резолвер с app.url_path_for
```

Если у приложения свой lifespan: `plugin.lifespan` композируется (`contextlib.AsyncExitStack`)
или `await plugin.startup()/shutdown()` вручную.

## Commit-safety и dispatch

- ORM-часть целиком из 22. Резолвер по умолчанию: `app.url_path_for(name, **params)` +
  `base_url`, объявление модели `indexnow_route = ("post_detail", {"slug": "slug"})` как
  альтернатива `get_indexnow_urls`.
- Collector scope: pure-ASGI middleware открывает `Collector.scope()` на запрос, после
  отправки ответа (после `await app(...)`) вызывает flush. Это после ответа клиенту (H06).
- Dispatcher: `AsyncDispatcher` (task на loop, `httpx.AsyncClient` из lifespan). Для
  production в README: `CallableDispatcher` на Celery/arq/taskiq. `BackgroundTasks` не
  используем: не переживает падение и не батчится между обработчиками.

## Key file

`@app.get("/{key}.txt", response_class=PlainTextResponse, include_in_schema=False)`, 404 для
чужого ключа.

## CLI

`python -m indexnowkit ...` из core. FastAPI-специфичных команд нет.

## Тесты

`httpx.AsyncClient(transport=ASGITransport(app))`, pytest-asyncio. Conformance H01–H06, A01–A14
через 22.
