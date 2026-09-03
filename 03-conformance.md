# 03. Conformance suite и mock-сервер

Один набор сценариев, который обязан проходить каждый core и каждый адаптер. Живёт в
`indexnowkit/spec`, публикуется как Docker image `ghcr.io/indexnowkit/mock-server` и как
JSON-fixtures.

## Mock-сервер

HTTP-сервер (Go, один бинарник), эмулирует `/indexnow`:

- Принимает GET и POST, валидирует по правилам протокола, отвечает кодом по сценарию.
- Сценарий выбирается заголовком `X-Mock-Scenario: <name>` или query `?scenario=`.
  Клиенты в тестах ставят endpoint `http://mock:8080/indexnow` и заголовок через
  `http.extra_headers` (опция core только для тестов, недокументированная публично).
- Записывает все запросы: `GET /_mock/requests` возвращает JSON-лог (метод, body, headers,
  timestamp). `DELETE /_mock/requests` очищает.
- Сценарии: `ok200`, `pending202`, `bad400`, `forbidden403`, `unprocessable422`,
  `ratelimit429-then-ok` (первые N запросов 429, потом 200; N в query), `timeout` (спит
  30 с), `flaky500-then-ok`.
- Также отдаёт `GET /{key}.txt` для сценария проверки ключа, если ключ в allowlist
  (`MOCK_KEYS=abc,def`).

## Сценарии core (обязательны для всех языков)

| ID | Сценарий | Ожидание |
|---|---|---|
| C01 | submit 1 URL | 1 POST, body `{host,key,urlList:[url]}`, без keyLocation |
| C02 | submit с keyLocation | body содержит keyLocation |
| C03 | 10 001 URL одного host | 2 POST: 10 000 + 1 |
| C04 | URL двух host | 2 POST, по одному на host |
| C05 | URL чужого host при `hosts` карте без него | отброшен с warning, POST не отправлен |
| C06 | дубликаты в одном вызове | дедуп, 1 URL в body |
| C07 | тот же URL дважды в пределах debounce | второй раз POST нет |
| C08 | тот же URL после истечения debounce | POST есть |
| C09 | ответ 202 | Result.status=pending, URL помечен отправленным |
| C10 | ответ 403 | Result.failed, retryable=false, лог error содержит `/{key}.txt` |
| C11 | ответ 422 | failed, retryable=false |
| C12 | ответ 429 в sync-режиме | failed, retryable=true, ретраев нет, исключение не брошено |
| C13 | ответ 429 в queue-режиме | ретрай с backoff, потом ok |
| C14 | timeout | failed, retryable=true, длительность ≤ timeout+1 s |
| C15 | ключ `abc` (короче 8) в конфиге | ConfigurationError при построении, не при submit |
| C16 | `enabled: false` | POST нет, лог debug |
| C17 | `dry_run: true` | POST нет, лог info с полным body |
| C18 | engines: [yandex, bing] | 2 POST на разные endpoint с одинаковым body |
| C19 | URL с `#fragment` и не-ASCII host | фрагмент удалён, host в punycode |
| C20 | `submit([])` | ничего не делает, без ошибки |
| C21 | генерация ключа | 32 символа hex, два вызова различаются |
| C22 | throttle 2 req/min, 3 батча | третий отложен ≥ до следующего окна (queue) |

## Сценарии адаптера (ORM)

| ID | Сценарий | Ожидание |
|---|---|---|
| A01 | создать сущность с атрибутом, commit | 1 URL в Collector, POST после commit |
| A02 | создать, rollback | POST нет |
| A03 | обновить | POST с тем же URL |
| A04 | удалить | POST с URL, вычисленным до удаления |
| A05 | вложенная транзакция, внешний rollback | POST нет |
| A06 | 3 сущности в одной транзакции | 1 POST с 3 URL |
| A07 | сущность без атрибута | ничего |
| A08 | `when` возвращает false (draft) | ничего |
| A09 | published → draft | POST (как deleted) если адаптер это поддерживает; иначе документировано |
| A10 | исключение в UrlResolver | лог error, транзакция пользователя не сломана |
| A11 | ошибка HTTP (mock 500) | ответ приложения 200, ошибка в логе |
| A12 | `on_fields: [title]`, изменено только `views` | POST нет |
| A13 | bulk-операции (QuerySet.update, DQL UPDATE, updateMany) | документировано: хуки не срабатывают; есть ручной `submit` |
| A14 | dispatch: queue | сообщение в очереди, воркер шлёт POST |

## Сценарии адаптера (HTTP/фреймворк)

| ID | Сценарий | Ожидание |
|---|---|---|
| H01 | GET `/{key}.txt` | 200, `text/plain`, тело = ключ |
| H02 | GET `/other.txt` | 404 (не отдаём произвольные файлы) |
| H03 | `serve_key_file: false` | 404 на `/{key}.txt` |
| H04 | `check` команда при доступном mock | exit 0, вывод содержит host и engine |
| H05 | `check` при 403 | exit 1, вывод содержит подсказку |
| H06 | submit во время HTTP-запроса, sync | POST уходит после отправки ответа клиенту (там, где платформа позволяет), иначе после обработчика |

## Реализация

- Fixtures хранятся как YAML в `spec/conformance/*.yaml` с полями `id`, `given`, `when`,
  `then`, `mock_scenario`. Языковые тест-раннеры читают YAML и генерируют тесты
  параметризованно (PHPUnit DataProvider, pytest.mark.parametrize, vitest each).
- CI каждого репозитория поднимает mock-сервер как service container.
- Бейдж в README: «Conformance: 22/22 core, 14/14 orm, 6/6 http».
