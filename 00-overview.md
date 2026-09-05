# 00. Обзор проекта

Статус: draft v0.1, 2026-09-03. Владелец: kolgarn.

## Цель

Семейство библиотек, которое за одну команду установки добавляет в приложение push-уведомления
поисковиков по протоколу IndexNow (Yandex, Bing, Naver, Seznam, Yep, Internet Archive, Amazon) при создании, изменении и
удалении контента. Целевой опыт разработчика: `composer require` / `pip install` / `npm i`,
одна строка конфигурации, один атрибут или миксин на модели. Всё остальное (ключ, батчинг,
дебаунс, отправка после commit, ретраи) библиотека делает сама.

Аналогия по DX: Gedmo DoctrineExtensions (`#[Gedmo\Timestampable]`) для Symfony,
`django-model-utils` для Django, Prisma client extensions для JS.

## Почему это нужно

- Существующие пакеты (см. 90-distribution.md, раздел «Конкуренты») в 90% случаев это
  обёртка над одним HTTP-вызовом. Никто системно не решает пост-commit отправку, дебаунс
  одного URL (Yandex просит не чаще раза в 10 минут), разбиение на 10 000, обработку 202/429.
- Ниши Symfony/Doctrine, Django, SQLAlchemy, Prisma, TypeORM, Nuxt, SvelteKit, NestJS,
  Rails, Spring, EF Core пустые (проверено 2026-09-03 по Packagist, PyPI, npm).
- Единый бренд и одинаковый README на все экосистемы дают SEO-эффект: запрос
  «<framework> indexnow» ведёт на наш пакет.

## Не-цели

- Google. Google не поддерживает IndexNow, sitemap ping выключен (404), Indexing API
  разрешён только для `JobPosting` и `BroadcastEvent`. Мы не обещаем Google. Опциональный
  адаптер Google Indexing API возможен только как отдельный пакет с явным типом контента
  (см. 91-roadmap.md, фаза 4).
- Генерация sitemap. Только интеграция: тот же хук может обновлять `lastmod` через
  существующий sitemap-пакет фреймворка.
- SaaS/дашборд. Только библиотеки.
- Краулинг сайта, проверка статуса индексации.

## Принципы

1. Core без зависимостей на фреймворк. Один core на язык. Адаптеры тонкие (50–200 строк).
2. Безопасность по умолчанию: отправка только после commit; дебаунс; никогда не ронять
   пользовательский запрос из-за ошибки IndexNow (ошибки логируются, не пробрасываются).
3. Честность: в документации явно написано, кто получает уведомления, а кто нет.
4. Один conformance test-suite на все адаптеры (03-conformance.md).
5. Нулевая магия в проде без явного opt-in: адаптер ничего не отправляет, пока не
   объявлена хотя бы одна модель и не задан ключ.
6. Идентичная терминология и структура конфигурации во всех языках (02-core-architecture.md).

## Бренд и именование

Решение открыто. Рабочее имя бренда: **`indexnow-kit`** (vendor `indexnowkit`).
Ограничение: «IndexNow» это имя протокола Microsoft, scope `@indexnow/*` на npm свободен,
но использовать его рискованно (trademark). Бренд обязан содержать слово `indexnow` в
имени пакета для поиска, но не быть только им.

Схема имён (обязательна для всех адаптеров):

| Экосистема | Core | Адаптер |
|---|---|---|
| PHP | `indexnowkit/core` | `indexnowkit/symfony-bundle`, `indexnowkit/laravel`, `indexnowkit/doctrine`, `indexnowkit/api-platform` |
| Python | `indexnowkit` | `indexnowkit-django`, `indexnowkit-sqlalchemy`, `indexnowkit-fastapi`, `indexnowkit-flask`, `indexnowkit-wagtail` |
| JS/TS | `@indexnowkit/core` | `@indexnowkit/prisma`, `@indexnowkit/typeorm`, `@indexnowkit/drizzle`, `@indexnowkit/next`, `@indexnowkit/nuxt`, `@indexnowkit/sveltekit`, `@indexnowkit/nestjs`, `@indexnowkit/payload`, `@indexnowkit/strapi` |
| Ruby | `indexnowkit` | `indexnowkit-rails` |
| Go | `github.com/indexnowkit/indexnow-go` | подпакеты `gorm`, `ent`, `httpkey` |
| Java | `dev.indexnowkit:indexnow-core` | `dev.indexnowkit:indexnow-spring-boot-starter` |
| .NET | `IndexNowKit` | `IndexNowKit.EntityFrameworkCore`, `IndexNowKit.AspNetCore` |

Ключевые слова в метаданных пакета всегда: `indexnow`, `seo`, `yandex`, `bing`, `<framework>`.

## Структура репозиториев

GitHub-организация `indexnowkit`. Один монорепозиторий на язык, релизы пакетов независимые
(subtree split для Packagist, changesets для npm, отдельные `pyproject` в workspace).

```
indexnowkit/php        packages/core, packages/symfony-bundle, packages/laravel, ...
indexnowkit/python     packages/indexnowkit, packages/indexnowkit-django, ...
indexnowkit/js         packages/core, packages/prisma, packages/next, ...
indexnowkit/ruby       indexnowkit.gemspec, rails/ (один gem с опциональным railtie)
indexnowkit/go         один модуль, подпакеты
indexnowkit/java       core, spring-boot-starter (Gradle multi-module)
indexnowkit/dotnet     src/IndexNowKit, src/IndexNowKit.EntityFrameworkCore, ...
indexnowkit/spec       эта спецификация + conformance fixtures + mock server (Docker image)
indexnowkit/.github    общие workflows, README-шаблон, issue templates
```

## Состав спецификации

- 01-protocol.md: факты о протоколе и политика обработки ответов.
- 02-core-architecture.md: языконезависимая архитектура core, конфиг, поведение.
- 03-conformance.md: общий тест-набор и mock-сервер.
- 10–14: PHP (core, doctrine, symfony-bundle, laravel, api-platform/CMS).
- 20–25: Python (core, django, sqlalchemy, fastapi, flask, wagtail).
- 30–39: JS/TS (core, prisma, typeorm, drizzle/mongoose/sequelize, next, nuxt, sveltekit/react-router, nestjs, payload, strapi/directus/sanity).
- 40–44: Ruby/Rails, Go, Java/Spring, .NET, прочее (Elixir, Rust, SSG, PHP CMS).
- 90-distribution.md: README-шаблон, каталоги, SEO пакетов, конкуренты.
- 91-roadmap.md: порядок выпуска, критерии готовности.

## Порядок реализации (кратко)

1. spec + mock server + conformance fixtures.
2. PHP core + Symfony bundle (Doctrine) + Laravel.
3. Python core + Django + SQLAlchemy/FastAPI.
4. TS core + Prisma + Next + Nuxt + TypeORM/NestJS.
5. Rails, Spring Boot, EF Core, Go.
6. CMS-адаптеры (Payload, Strapi, Wagtail, API Platform) и Google Indexing API (опционально).
