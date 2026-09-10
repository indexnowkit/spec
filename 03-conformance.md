# 03. Conformance suite и mock-сервер

Один набор сценариев, который обязан проходить каждый core и каждый адаптер. Идентификаторы C01–C22, A01–A21 (+A05b, A05c, A10b),
S01–S08, H01–H06 **заморожены** как кросс-языковой контракт (спека 17 §7); новый сценарий — новый номер за диапазоном или
вариант с суффиксом (`H01b`), удаление — никогда. Реализация — на язык (§«Тест-кит»), контракт — этот текст плюс два файла
схем, которые `indexnowkit/spec` держит рядом: `check.schema.json` (`check --json`) и `status.schema.json` (`status --json`) — копии
`php/packages/console/docs/check.schema.json` и `php/packages/history/docs/status.schema.json` (решение спеки 26 §9.7).

## Mock-сервер

Контракт (реализация — своя на язык: PHP `php/packages/testing/resources/mock-server/router.php`, Python
`indexnowkit.testing.mock_server.MockIndexNowServer` на `http.server` в процессе; Go-бинарник и Docker-образ из первой редакции
не нужны — решение спеки 26 §9.8). HTTP-сервер эмулирует `/indexnow`:

- Принимает GET и POST, валидирует по правилам протокола, отвечает кодом по сценарию.
- Сценарий выбирается заголовком `X-Mock-Scenario: <name>` или query `?scenario=`.
  Клиенты в тестах ставят endpoint `http://mock:8080/indexnow` и заголовок через
  `http.extra_headers` (опция core только для тестов, недокументированная публично).
- Записывает все запросы: `GET /_mock/requests` возвращает JSON-лог (метод, body, headers,
  timestamp). `DELETE /_mock/requests` очищает.
- Сценарии (имена и коды — контракт): `ok200`, `pending202`, `bad400`, `forbidden403`, `unprocessable422`, `ratelimit429`
  (`Retry-After: 2`), `ratelimit429-then-ok` (первые N запросов 429 с `Retry-After: 1`, потом 200; N в query `n`), `flaky500-then-ok`
  (первые N — 503), `timeout` (спит 30 с в PHP; Python-реализация — параметр, чтобы тест на `http.timeout` шёл секунды, не
  полминуты), неизвестный сценарий — 400. Валидация POST до сценария: тело без `host`/`key`/`urlList` — 400, больше 10 000 URL — 400,
  URL не с `host` — 422, метод не GET/POST — 405.
- Также отдаёт `GET /{key}.txt` для сценария проверки ключа, если ключ в allowlist (`MOCK_KEYS=abc,def`), и
  `GET /large-document.xml[.gz]` (>100 КБ, регрессия усечения тела GET).

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

## Сценарии адаптера (модель правил)

Проверяют модель `UrlRule` из 02. Обязательны для адаптера, читающего правила с модели.

| ID | Сценарий | Ожидание |
|---|---|---|
| A15 | класс с тремя правилами (`route`, второй `route` с `when`, `urls`), обновление | один POST со всеми URL применимых правил, дедуплицированными |
| A16 | у сущности `when` второго правила `true → false`, первого — без изменений | оба URL в одном flush: URL второго правила как `deleted` (вычислен до записи), URL первого как `updated` |
| A17 | `when: 'isPublished'` (геттер) при поле `published` в change set, `true → false` | классифицируется как `deleted`, не как `updated`: поле находится по конвенции |
| A18 | удаление объекта, у которого `when` ложен (черновик) | POST нет |
| A19 | `#[IndexNow(via: 'post')]` на комментарии, комментарий изменён | POST с URL правил связанного поста, событие `updated`, имя правила содержит цепочку `via:post -> ...` |
| A20 | изменение to-many коллекции владельца (`post.tags`), сам владелец не менялся | POST с URL правил владельца (изменение коллекции не входит в его change set) |
| A21 | изменено поле, которое читает параметр маршрута (slug), страница была публичной | старые URL правила (по прежним значениям change set) как `deleted` и новые как `updated` в одном flush; поле `readonly` — только новые URL, debug в логе |

## Сценарии адаптера (HTTP/фреймворк)

| ID | Сценарий | Ожидание |
|---|---|---|
| H01 | GET `/{key}.txt` | 200, `text/plain`, тело = ключ |
| H02 | GET `/other.txt` | 404 (не отдаём произвольные файлы) |
| H03 | `key_file.enabled: false` (`serve_key_file` — deprecated-псевдоним в PHP) | 404 на `/{key}.txt` |
| H04 | `check` команда при доступном mock | exit 0, вывод содержит host и engine |
| H05 | `check` при 403 | exit 1, вывод содержит подсказку |
| H06 | submit во время HTTP-запроса, sync | POST уходит после отправки ответа клиенту (там, где платформа позволяет), иначе после обработчика |

## Сценарии хранилища отправок

S01–S08 (`SubmissionStoreInterface`, `core/docs/submission-store.md`): запись и чтение `Result` с временем, newest-first, фильтры по
host и status (skipped — тоже записи), `lastFor(url)` — последняя запись с URL любого статуса, `recent(limit)`, несколько URL одного
`Result` — одна запись, пустой стор — ничего, `purge()` там, где поддерживается. Кит — `SubmissionStoreConformanceTestCase` (PHP),
`SubmissionStoreConformance` (Python).

## Тест-кит для адаптеров

PHP: абстрактные PHPUnit-кейсы в `indexnowkit/testing` (`IndexNowKit\Testing\Conformance`, с 0.7.0 — не в core), покрыты
BC-обещанием (`testing/docs/bc.md`: методы драйвера растут только с реализацией по умолчанию, сценарий только добавляется).
Python: pytest-классы-миксины в `indexnowkit[testing]` (`indexnowkit.testing.conformance`: `CoreConformance`, `OrmConformance`,
`SubmissionStoreConformance`, `KeyFileAssertions`, `CheckOutputAssertions`, `ReadmeAssertions`), тот же драйвер — спека 20 §3.1.

- `CoreConformanceTestCase` — C01, C03, C04, C06, C09–C12, C14, C19, C20 против фасада, собранного контейнером
  адаптера (адаптер отдаёт фасад, `FakeTransport` и, опционально, второй настроенный host). Сценарии, требующие
  особой конфигурации (dry_run, enabled: false, engines, окна debounce, throttle), остаются в тестах core.
- `OrmConformanceTestCase` — A01–A21 (+A05b вложенный commit, +A05c откат к savepoint) через драйвер, который
  реализует адаптер: транзакционные глаголы его слоя данных (`begin/commit/rollback`), конец единицы работы
  (`flush`, `collectedCount`) и фикстуры с фиксированными формами правил (post с `when` и `fields`, multi-post с
  тремя правилами и getter-`when`, categorized post с `via` и to-many коллекцией, category, untracked, broken,
  bad attribute; `update/delete/attachTag/bulkUpdateTitle`). URL-конвенции переопределяемы. Эталонные драйверы:
  `packages/doctrine/tests/OrmConformanceTest.php`, `packages/laravel/tests/Conformance/OrmConformanceTest.php`.
  Symfony-бандл гоняет A01/A02/A04 функционально поверх Doctrine (`tests/Functional/*`).

## Реализация

- **Сценарии — абстрактные тест-кейсы на языке, не YAML** (первая редакция обещала `spec/conformance/*.yaml` с параметризацией;
  в PHP YAML не появился, и правильно: сценарий A16 «переход `when` у одного правила при неизменном другом» не выражается данными
  без интерпретатора). Каждый сценарий — один метод кита с идентификатором в имени/аннотации (`#[TestDox('A16 …')]` в PHP,
  `def test_a16_…` в Python); тест полноты (`ConformanceIdsTest` PHP, `test_ids` Python) проверяет: каждый id ровно один раз,
  диапазон заморожен, каждый фреймворк-адаптер несёт H01–H06 (адаптеры находятся по файловой системе, не по списку).
- Mock-сервер — в процессе тестов (PHP `php -S`, Python `ThreadingHTTPServer` на порту 0); service container в CI не нужен.
- Бейдж в README: «Conformance: 22/22 core, 21/21 orm, 6/6 http, 8/8 store». Адаптер, читающий правила с модели,
  добавляет к нему A15–A21; адаптер без такой модели (например, чисто транспортный) объявляет их
  неприменимыми в README с обоснованием. Сценарий, неприменимый к фреймворку, называется в README, не пропускается молча.
