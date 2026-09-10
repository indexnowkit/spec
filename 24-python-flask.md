# 24. Python: `indexnowkit-flask` (волна P, шаг 5, по решению)

Статус: **переписана 2026-09-10** (прежняя редакция 2026-09-03 заменена). Тонкая обёртка над спеками 20 и 22 в конвенциях Flask
(extension с `init_app`). Решения — §9; ставится ли в волну вообще — спека 26 §9.5.

## 0. Цель и границы

Что добавляет пакет: (1) extension `IndexNowKit()`/`init_app(app)` с конфигурацией из `app.config["INDEXNOW"]` (вложенный словарь
формы core) или плоских `INDEXNOW_*` ключей `app.config` тем же правилом, что окружение; (2) маршрут `/<key>.txt`; (3) scope
коллектора на запрос (`before_request`) и flush после ответа через **WSGI `close()`** (`wsgi.CollectorMiddleware` core), потому что
ни один хук Flask не выполняется после отправки ответа (факт §1.2); (4) `RouteUrlResolver` над `url_for(endpoint, _external=True,
**params)` с `app.app_context()` вне запроса (`SERVER_NAME` или `base_url`); (5) `flask indexnow <cmd>` — click-группа над раннерами
core; (6) `init_sqlalchemy(db)` → `install(db.session)` спеки 22. Границы: Quart — по спросу (ASGI core покрывает).

## 1. Факты (проверены 2026-09-10)

1. Flask 3.1.3 (2026-02-19, ≥3.9; `click>=8.1.3`, `werkzeug>=3.1`); Flask-SQLAlchemy 3.1.1 (**2023-09-11**, три года без релиза;
   `sqlalchemy>=2.0.16`); click 8.5.0 (2026-08-26).
2. **Хуки** (flask.palletsprojects.com/en/stable/api): `before_request`, `after_request(response)` (не вызывается при исключении),
   `teardown_request(exc)` и `teardown_appcontext(exc)` выполняются **и при исключении**, не должны бросать, значения игнорируются;
   **ни один не выполняется после отправки ответа**. `Flask.cli` — `click.Group`; `AppGroup`/`with_appcontext` — в `flask.cli`.
3. WSGI (PEP 3333): сервер вызывает `close()` итерируемого тела после отдачи — единственная точка «после ответа» под WSGI.
4. `url_for(..., _external=True)` вне запроса требует `SERVER_NAME` (и `PREFERRED_URL_SCHEME`) в `app.config`.

## 2. Принципы

≤ 300 строк; Flask-конвенции (extension, `app.extensions["indexnowkit"]`, `current_app`); всё общее — core `wsgi`; ноль текстов `check`
своих; `flask indexnow` = те же команды и опции (`Definitions` → click).

## 3. Дизайн

```python
from indexnowkit_flask import IndexNowKit
indexnow = IndexNowKit()
indexnow.init_app(app)            # app.config["INDEXNOW"] | INDEXNOW_* ключи; /<key>.txt; wsgi CollectorMiddleware; flask indexnow …
indexnow.init_sqlalchemy(db)      # спека 22 на db.session с роутером url_for
```

- `init_app`: `ConfigFactory.load()` над `app.config` (+ `Config.mapping_from_env()` поверх), `app.wsgi_app = CollectorMiddleware(app.wsgi_app, kit)`,
  `app.add_url_rule("/<key>.txt", "indexnowkit.key_file", view)` (404 через `KeyFileResponder`), `app.cli.add_command(group)`,
  `app.extensions["indexnowkit"] = self`; несколько приложений — состояние по `app`, не в extension.
- `dispatch`: `sync` (в `close()` после ответа), `thread`, `callable` (Celery `shared_task`, RQ), `none`; `debounce.store`: `memory`,
  `sqlite`, объект `get/set/add` (Flask-Caching даёт `cache.cache` с `add`) через `IndexNowKit(cache=…)`.
- `flask indexnow check|config|submit|submit-objects|explain|key generate|key file|sitemap|history|status` — `click` команды из
  `Definitions` (конвертер `apply_to_click(cmd)` в адаптере: ~40 строк), `with_appcontext`; `SubjectLoader` — по классу модели
  SQLAlchemy (`db.session.get`).
- `check` адаптера: `wiring.wsgi` (middleware обёрнут; предупреждение, если `app.wsgi_app` заменили после `init_app`), `router.server_name`
  (`SERVER_NAME` не задан и `base_url` пуст при правилах `route`).

## 4. BC и версии

`indexnowkit-flask` 0.1.0; `flask>=3.0`, `optional-dependencies.sqlalchemy = ["indexnowkit-sqlalchemy>=0.1,<0.2", "flask-sqlalchemy>=3.1"]`;
`requires-python >=3.11`. Публичное — `IndexNowKit` extension и команды.

## 5. Тесты

`app.test_client()` — H01–H06 (H06: `close()` итератора тела вызывается тест-клиентом Werkzeug — проверить; иначе явный `response.close()`);
A01–A21 через драйвер спеки 22 с `url_for`-роутером и `app.app_context()`; `test_cli.py` (`CliRunner`, `--json`); `test_config.py`
(вложенный словарь vs плоские ключи vs env); `test_readme.py`. Матрица: Flask 3.1 × (3.11–3.14), `lowest` 3.0.

## 6. Документация

README EN/RU (install, `app.config`, модель, `flask indexnow check`, Celery, AI-notes), `docs/`: `configuration.md`, `dispatch.md`,
`sqlalchemy.md`, `testing.md`, `bc.md`.

## 7. Риски

1. **Werkzeug test client и `close()`** — вызывается ли `close()` тела в `test_client().get()`? Проверить первым; H06 зависит.
2. **`url_for` вне запроса и blueprints с `subdomain`** — `SERVER_NAME` обязателен; `check` строка.
3. **Flask-SQLAlchemy 3.1.1 без релизов** — совместимость с SQLAlchemy 2.1 не заявлена; в матрице `experimental`.

## 8. Не делать

`models_committed`/`SQLALCHEMY_TRACK_MODIFICATIONS`; `after_request` как точка отправки (до ответа); Quart-вариант; свой парсер конфигурации.

## 9. `[решение]` — рекомендации

1. **Пакет ставится последним в волне и только если user подтвердит** (спека 26 §9.5): ценность — SEO-имя и `flask indexnow` CLI;
   всё техническое покрывает core `wsgi` + рецепт. Рекомендация: **да, делать** (1–2 дня), после FastAPI.
2. **Flush через WSGI `close()`, не `teardown_request`** — да (единственная точка после ответа).
3. **Конфигурация: вложенный `app.config["INDEXNOW"]` первичен, плоские `INDEXNOW_*` — как окружение** — да.

## 10. Definition of Done

Приложение из README с Flask-SQLAlchemy: `flask indexnow check` зелёный; создание объекта → POST после ответа; H01–H06, A01–A21;
mypy strict; README EN/RU; docs.
