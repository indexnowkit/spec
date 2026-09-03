# 24. Python: `indexnowkit-flask`

Flask ≥ 3.0, Flask-SQLAlchemy ≥ 3.1 (optional).

## Установка

```python
from indexnowkit_flask import IndexNowExtension
indexnow = IndexNowExtension()
indexnow.init_app(app)     # читает app.config["INDEXNOW_*"], роут /<key>.txt, request hooks
indexnow.init_sqlalchemy(db)   # db.session → install() из 22
```

Конфиг: `INDEXNOW_KEY`, `INDEXNOW_BASE_URL`, `INDEXNOW_DISPATCH`... в `app.config`
(Flask-конвенция плоских UPPER-ключей с префиксом).

## Commit-safety и dispatch

- Не использовать `models_committed` Flask-SQLAlchemy (требует `SQLALCHEMY_TRACK_MODIFICATIONS`,
  deprecated-подход). Только события SQLAlchemy из 22 на `db.session`.
- Collector scope: `before_request` открывает scope в `flask.g`, `teardown_request` делает
  flush (после ответа в WSGI не гарантировано; используем `ThreadDispatcher` по умолчанию,
  `sync` опционально).
- Резолвер по умолчанию: `url_for(endpoint, _external=True, **params)` требует app context;
  `install` оборачивает вызов в `app.app_context()` при работе вне запроса. Нужен `SERVER_NAME`
  в конфиге для внешних URL вне запроса, иначе fallback на `base_url`.

## Key file

`app.add_url_rule("/<key>.txt", view_func=...)`, сравнение с ключом, 404 иначе.

## CLI

`flask indexnow key|check|sitemap` через `app.cli.group("indexnow")`, обёртки над core CLI.
