# Спецификация indexnowkit

Семейство IndexNow-библиотек для основных языков и фреймворков. Читать по порядку:
00 → 01 → 02 → 03, затем раздел своей экосистемы, затем 90–91.

| Файл | Тема |
|---|---|
| 00-overview.md | Цели, принципы, бренд, структура репозиториев |
| 01-protocol.md | Протокол IndexNow, коды ответов, политика ретраев |
| 02-core-architecture.md | Компоненты core, единая схема конфига, объявление модели |
| 03-conformance.md | Mock-сервер и общий тест-набор |
| 10–14 | PHP: core, Doctrine, Symfony, Laravel, API Platform/CMS |
| 20–25 | Python: core, Django, SQLAlchemy, FastAPI, Flask, Wagtail |
| 30–39 | JS/TS: core, Prisma, TypeORM, Drizzle/Mongoose/Sequelize, Next, Nuxt, SvelteKit/RR7, NestJS, Payload, Strapi/Directus/Sanity |
| 40–44 | Rails, Go, Spring Boot, .NET, прочее |
| 90-distribution.md | README-шаблон, каталоги, конкуренты |
| 91-roadmap.md | Фазы, DoD, открытые решения |

## Сводка commit-safety по экосистемам

| Хук нативно после commit | Требует staging + сигнал commit | Нет хуков, только обёртка |
|---|---|---|
| Rails `after_*_commit`, Sequelize `transaction.afterCommit`, ent `OnCommit`, EF Core (SaveChanges + transaction interceptor), Laravel `ShouldHandleEventsAfterCommit`, Django `transaction.on_commit` | Doctrine (DBAL driver middleware), SQLAlchemy (`before_flush` + `after_commit`), TypeORM (`afterTransactionCommit`), JPA (`@TransactionalEventListener`), GORM (обёртка `Transaction` или outbox), Mongoose (session staging), Payload (`afterOperation`) | Prisma (`indexNowTransaction`), Drizzle (`inx.transaction`), Ecto, SeaORM |

## Открытые вопросы для проверки перед реализацией

1. Prisma: способ определить interactive-transaction контекст внутри `$extends`.
2. TypeORM: `afterTransactionCommit` при savepoint-вложенности.
3. Payload 3: порядок `afterChange` → commit → `afterOperation`.
4. Strapi 5: Document Service middleware относительно commit транзакции.
5. `@nuxtjs/seo`: появилась ли автоотправка.
6. Django CMS: имя сигнала публикации.
7. Maven Central: прямая проверка отсутствия `indexnow` артефактов.
8. Trademark «IndexNow» и свободность бренда `indexnowkit` (домен, org на GitHub/npm/Packagist/PyPI).
