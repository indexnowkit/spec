# 91. Roadmap и критерии готовности

## Фазы

| Фаза | Что | Зачем первым |
|---|---|---|
| 0 | `indexnowkit/spec`: эта спецификация, mock-сервер, conformance YAML, README-шаблон, docs-сайт скелет | без этого адаптеры разъедутся |
| 1 | PHP: core, symfony-bundle (Doctrine), laravel | самая пустая ниша (Symfony) + самый большой спрос (Laravel); автор знает PHP |
| 2 | Python: core, django, sqlalchemy, fastapi, wagtail (обёртка над core вместо wagtail-indexnow) | Django ниша пустая, аудитория большая |
| 3 | JS: core, prisma, next, nuxt, typeorm, nestjs, drizzle, sveltekit, payload, strapi | много мелких адаптеров, ценность в покрытии |
| 4 | Rails, Spring Boot starter, EF Core, Go (gorm) | закрыть матрицу |
| 5 | Опционально: `google-indexing-api` адаптер (только JobPosting/BroadcastEvent, с явным `content_type`), sitemap-lastmod интеграции, API Platform, Drupal-модуль | по спросу |

## Definition of Done для пакета

- Conformance: все применимые сценарии из 03 зелёные в CI с mock-сервером.
- README по шаблону, RU-версия для PHP/Python/JS.
- Команда `check` реализована.
- Матрица версий фреймворка в CI (минимум 2 последних major/LTS).
- Static analysis на максимуме (phpstan level 9 / mypy strict / tsc strict).
- Опубликован в реестр, зарегистрирован минимум в двух каталогах из 90.
- Ссылка добавлена в таблицу семейства во всех остальных README.

## Метрики успеха (12 месяцев)

- Позиция 1–3 по запросу «<framework> indexnow» для 6 фреймворков.
- Суммарно ≥ 50k загрузок/мес по семейству.
- ≥ 1 внешний контрибьютор на адаптер.

## Решения, принятые 2026-09-03

- Бренд `indexnowkit`. Свободно: GitHub org, Packagist vendor, PyPI, RubyGems, домены
  indexnowkit.dev/.com/.io (нет DNS). npm-профиль не проверен (403 от бот-защиты): проверить
  при `npm org create`.
- GitHub org + монорепо на язык.
- Порядок: PHP (core → doctrine → symfony-bundle → laravel) → JS (core → next → prisma) →
  Python (core → django → sqlalchemy → fastapi). Остальное по спросу.
- Версии: PHP 8.2, Node 20, Python 3.11, Django 4.2+. Dispatch авто: queue при наличии
  Messenger/Laravel Queue, иначе sync после ответа.
- Документация EN основная, README.ru.md, MIT, VitePress на indexnowkit.dev.
- Старт: сразу PHP core + Symfony bundle; mock-сервер сначала как PHP-тестовый (встроенный
  сервер в тестах), выделение в Go-бинарник позже.
- WordPress вне scope подтверждено.

## Статус реализации (2026-09-03)

- `php/packages/core`: готов. Conformance C01–C22 (C13 в бандле), phpstan 9, PHP 8.2–8.4.
- `php/packages/doctrine`: готов. A01–A14 на sqlite, ORM 3 + DBAL 4 (DBAL 3 в CI-матрице).
- `php/packages/symfony-bundle`: готов. H01–H03, A01/A02/A04 функционально, Messenger (C13),
  команды `indexnow:key:generate|check|submit|submit-entity|sitemap`, Web Profiler панель, Flex-рецепт в `recipe/`,
  split-workflow для Packagist (`.github/workflows/split.yml`, deploy-ключи в секретах SPLIT_SSH_KEY_*).
- Опубликовано 2026-09-03: GitHub org `indexnowkit` (репо php, php-core, php-doctrine, php-symfony-bundle, spec), npm org
  `indexnowkit`, Packagist `indexnowkit/core|doctrine|symfony-bundle` 0.1.0. E2E-установка в чистый Symfony 7.4 skeleton
  с DoctrineBundle 3.3 проверена. Автообновление Packagist: GitHub-хуки установлены самим Packagist (OAuth-доступ приложения к org выдан, sync через packagist.org/trigger-github-sync/).
- Не сделано: PR Flex-рецепта в recipes-contrib (отложено пользователем), Laravel.

## Открытые решения

1. Trademark «IndexNow» у Microsoft: допустимо ли в имени бренда. Проверить до первой публикации.
2. Wagtail: PR в `wagtail-indexnow` или свой пакет. Дефолт: PR + обёртка через 30 дней.
3. Google Indexing API адаптер (JobPosting/BroadcastEvent): только по спросу.
