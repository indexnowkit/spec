# 20. Python: core `indexnowkit`

Реализует контракты из 02. Пакет `indexnowkit` на PyPI, модуль `indexnowkit`.

## Требования

- Python ≥ 3.11 (3.10 теряет security support в октябре 2026).
- Зависимости core: **ноль**. HTTP через `urllib.request` (sync). Async-транспорт на `httpx`
  как extra `indexnowkit[httpx]`.
- `py.typed`, mypy strict, ruff.
- Build: hatchling, PEP 621, monorepo `indexnowkit/python` с workspace (uv).

## Публичный API

```python
from indexnowkit import IndexNow, Config, Engine, Result

inx = IndexNow(Config(key="...", base_url="https://www.example.com", engines=[Engine.API]))
results: list[Result] = inx.submit(["https://www.example.com/a", "/relative/ok"])
inx.submit_object(post)               # через UrlResolver из registry
await inx.asubmit([...])              # если установлен httpx
```

- Относительные URL резолвятся относительно `base_url`. Без `base_url` относительный URL
  это `ValueError` (программная ошибка).
- `Result` dataclass frozen: `engine, host, url_count, status: Literal['ok','pending','failed'], http_code, error, retryable`.

## Компоненты (имена модулей)

| Модуль | Контракт из 02 |
|---|---|
| `indexnowkit.client` | `Client`, `Transport` protocol (`post(url, json, headers, timeout) -> (status, body)`), `UrllibTransport`, `HttpxTransport` |
| `indexnowkit.key` | `KeyProvider` protocol, `StaticKeyProvider`, `MapKeyProvider`, `generate_key(length=32)` через `secrets.token_hex` |
| `indexnowkit.url` | `normalize(url) -> str`, `host_of(url)`, `UrlResolver` protocol `resolve(obj, event) -> Iterable[str]`, `Event = Literal['created','updated','deleted']` |
| `indexnowkit.collector` | `Collector` на `contextvars.ContextVar[set[str] | None]`; `add`, `drain`, `scope()` context manager |
| `indexnowkit.debounce` | `DebounceStore` protocol, `MemoryDebounceStore` (dict + TTL, потокобезопасный через `threading.Lock`) |
| `indexnowkit.submitter` | `Submitter.submit(urls) -> list[Result]`: debounce → group by host → chunk → throttle → client → mark |
| `indexnowkit.dispatcher` | `Dispatcher` protocol `dispatch(urls)`, `SyncDispatcher`, `ThreadDispatcher` (daemon thread, для сред без очереди), `CallableDispatcher(fn)` для Celery/RQ |
| `indexnowkit.registry` | `Registry.register(cls, url=..., when=..., events=..., on_fields=...)`, `resolver_for(obj)` по MRO |
| `indexnowkit.config` | `Config` dataclass, `Config.from_mapping(dict)`, `Config.from_env(prefix="INDEXNOW_")`, валидация в `__post_init__` |
| `indexnowkit.errors` | `IndexNowError`, `ConfigurationError`, `InvalidUrlError` |
| `indexnowkit.sitemap` | `iter_sitemap_urls(sitemap_url, transport) -> Iterator[str]` (stdlib `xml.etree`, sitemap index рекурсивно, gzip) |
| `indexnowkit.cli` | `python -m indexnowkit key|check|sitemap <url>` (argparse), конфиг из env |

## Registry и объявление модели

Единый механизм для всех адаптеров:

```python
from indexnowkit import registry

class IndexNowMixin:                      # маркер + дефолтный резолвер
    def get_indexnow_urls(self) -> list[str]:
        return [self.get_absolute_url()]  # Django-конвенция; для SQLAlchemy переопределить
    indexnow_when: Callable[[Any], bool] | None = None

registry.register(Post, url=lambda p: f"/posts/{p.slug}", when=lambda p: p.is_published,
                  events={"created", "updated", "deleted"}, on_fields={"slug", "title", "body"})
```

Порядок резолва: явная регистрация в registry → `get_indexnow_urls()` → `get_absolute_url()`.
`on_fields` реализуется адаптером (только он знает diff).

## Логирование

`logging.getLogger("indexnowkit")`. Уровни по 02. Ключ в логах маскируется
`key[:4] + "…"`.

## Conformance

`tests/conformance/test_core.py` читает `spec/conformance/core.yaml`, параметризует через
`pytest.mark.parametrize`. Mock-сервер: fixture поднимает Docker или использует
`INDEXNOW_MOCK_URL` из env в CI.
