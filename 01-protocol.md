# 01. Протокол IndexNow и политика обработки ответов

Источники: https://www.indexnow.org/documentation, https://yandex.ru/support/webmaster/ru/indexnow/reference,
https://www.indexnow.org/searchengines. Проверено 2026-09-03.

## Endpoint'ы

| Endpoint | Роль |
|---|---|
| `https://api.indexnow.org/indexnow` | Общий. Раздаёт всем участникам. **Дефолт для всех пакетов.** |
| `https://yandex.com/indexnow` | Yandex напрямую |
| `https://www.bing.com/indexnow` | Bing напрямую |
| `https://searchadvisor.naver.com/indexnow` | Naver |
| `https://search.seznam.cz/indexnow` | Seznam |
| `https://indexnow.yep.com/indexnow` | Yep |

Участники обязаны делиться полученными URL между собой, поэтому одного POST на
`api.indexnow.org` достаточно. Список endpoint'ов в core объявляется как enum/константы
`Engine`, конфигурация `engines: ['api']` по умолчанию, можно перечислить несколько
(тогда один и тот же батч отправляется в каждый).

Google не участник. Amazon указан участником, но публичного endpoint не публикует.

## Запросы

GET (один URL):

```
GET https://api.indexnow.org/indexnow?url=<urlencoded>&key=<key>[&keyLocation=<urlencoded>]
```

POST (1..10 000 URL, все с одного host):

```
POST https://api.indexnow.org/indexnow
Content-Type: application/json; charset=utf-8

{"host":"www.example.com","key":"<key>","keyLocation":"https://www.example.com/<key>.txt","urlList":["https://www.example.com/a","https://www.example.com/b"]}
```

Правила:
- `host` без схемы и пути. URL в `urlList` обязаны принадлежать этому host (иначе 422).
  Схемы http и https в одном батче допустимы.
- Core всегда использует POST, даже для одного URL (единый код, единый лог). GET оставлен
  как опция `transport.method: 'get'` для отладки.
- URL перед отправкой нормализуется (RFC 3986), фрагмент `#...` отбрасывается, IDN host
  переводится в punycode.
- Батч режется по host: разные host = разные POST. Внутри host режется по 10 000
  (конфигурируемо `batch.maxUrls`, дефолт 10 000, верхняя граница 10 000).

## Ключ

- Длина 8..128, символы `[A-Za-z0-9-]`. Core генерирует 32 символа hex из CSPRNG.
- Файл `https://<host>/<key>.txt` с телом ровно `<key>` (UTF-8, без BOM, завершающий
  перевод строки допустим, но core отдаёт без него). `Content-Type: text/plain`.
- Альтернатива `keyLocation`: файл в поддиректории, тогда валидны только URL с этим
  префиксом. Core поддерживает, адаптеры по умолчанию отдают в корне.
- Один ключ на host. Поддомены это разные host, у каждого свой ключ. Core поддерживает
  карту `hosts: {'example.com': 'KEY1', 'blog.example.com': 'KEY2'}` плюс `key` как дефолт
  для единственного host.
- Ключ хранится в env (`INDEXNOW_KEY`), никогда не коммитится. Адаптеры отдают файл ключа
  динамическим роутом `GET /{key}.txt`, а не физическим файлом, чтобы ротация ключа была
  сменой env.

## Ответы и политика core

| Код | Значение | Действие core |
|---|---|---|
| 200 | принято | success; пометить URL как отправленные (для дебаунса) |
| 202 | принято, ключ ещё не проверен | success-pending; лог `info`; пометить как отправленные; **ретрай не нужен** |
| 400 | невалидный формат | error `InvalidRequest`; лог `error` с телом; не ретраить |
| 403 | ключ не найден/не совпал | error `InvalidKey`; лог `error` с подсказкой про файл ключа; не ретраить; счётчик ошибок для health-check |
| 405 | не GET/POST | баг клиента, не ретраить |
| 422 | URL не с того host / keyLocation невалиден | error `UnprocessableUrls`; лог `warning`; не ретраить |
| 429 | rate limit по IP | error `RateLimited`; ретрай с экспоненциальной задержкой (base 60 s, max 3 попытки в async-режиме; в sync-режиме 0 ретраев, только лог) |
| 5xx / timeout / network | временная ошибка | ретрай как для 429, но base 5 s |

Ретраи выполняются только в async dispatcher (очередь). В sync-режиме (fire-and-forget в
конце запроса) любая ошибка только логируется, чтобы не блокировать HTTP-ответ.

Тело ответа у `api.indexnow.org` обычно пустое. Core читает первые 2 KB для лога и не
парсит его.

## Ограничения частоты

- Yandex: один и тот же URL не чаще чем раз в 10 минут (рекомендация). Core: дебаунс
  `debounce.perUrl = 600 s` по умолчанию, хранилище дебаунса pluggable (memory, cache
  фреймворка, таблица).
- Нет документированного лимита запросов в минуту. 429 срабатывает по IP. Core:
  `throttle.maxRequestsPerMinute = 60` для async dispatcher, простой token bucket.
- Слать только новые/изменённые/удалённые URL. Массовую отправку всего сайта (sitemap)
  оставить явной CLI-командой `submit-sitemap`, никогда не автоматической.

## Удаления и редиректы

- Удалённую страницу отправлять так же (движок увидит 404/410). Адаптеры на событие
  delete отправляют URL, вычисленный **до** удаления записи (URL надо резолвить в
  pre-remove, а отправлять в post-commit).
- 301/302 отправлять можно, core не проверяет.

## Что не делает протокол

- Не гарантирует индексацию. Не возвращает статус индексации. Не сообщает, был ли URL
  уже известен.
- Нет идемпотентности по body: повтор одного батча это повтор. Отсюда дебаунс на нашей
  стороне.
