# Спецификация indexnowkit

Семейство IndexNow-библиотек для основных языков и фреймворков. Читать по порядку:
00 → 01 → 02 → 03, затем раздел своей экосистемы, затем 90–91.

| Файл | Тема |
|---|---|
| 00-overview.md | Цели, принципы, бренд, структура репозиториев |
| 01-protocol.md | Протокол IndexNow, коды ответов, политика ретраев |
| 02-core-architecture.md | Компоненты core, единая схема конфига, объявление модели |
| 03-conformance.md | Общий тест-набор (C/A/S/H — замороженные идентификаторы, абстрактные киты на язык, не YAML), контракт mock-сервера, схемы `check`/`status --json` |
| 10–15 | PHP: core, Doctrine, Symfony, Laravel, API Platform/CMS, Yii2/Yii3 |
| 16-core-0.4-adapter-kit.md | PHP core 0.4/0.5 «adapter kit»: пакет `indexnowkit/sitemap`, `Adapter\ConfigFactory`, фабрики, `Adapter\Services`, общие блоки адаптеров |
| 17-php-family-1.0-readiness.md | PHP к 1.0: пакеты `testing`/`console`/`history`/`verify`, DX для людей и AI-ассистентов, эксплуатация (`check --json --strict`, стейджинг, ротация), SEO-честность, дистрибуция |
| 18-php-shared-console-commands.md | Волна L (после Yii3, до Битрикса): команды symfony/console живут в `console`/`sitemap`/`history`, адаптеры (бандл, Yii3) только регистрируют их; `ConfigSourceInterface`; PSR-15 `Key\KeyFileRequestHandler` в core; фаза B `bin/indexnow` — волна Битрикса |
| 19-php-leftovers-and-psr.md | Волна M (аудит после L): замер сходства всех одноимённых классов адаптеров, 12 гипотез с цифрами (Laravel регистрирует классы `indexnowkit/console` — спека 18 §9 ошиблась; сэмплер ×4, `LocalesCheck`, `BatchingDispatcher`, `RouteOrigin`, `isShared()`), ревизия PSR-1…20 по тексту стандартов, `[решение]` |
| 19b-php-cli-phar-action.md | Волна N (фаза B спеки 18): пакет `indexnowkit/cli` — `indexnow` без фреймворка (cron на любой CMS, Битрикс через штатный sitemap), файл состояния sqlite (дебаунс, история, `--new-only`), PHAR (Box), Docker-образ GHCR, GitHub Action на образе; факты со строками, риски проверены, `[решение]` |
| 20–25 | Python (переписаны 2026-09-10 в формате 19b): core `indexnowkit` — один дистрибутив с sitemap/verify/history/CLI и extra `[testing]`; Django (сигналы + `on_commit`, `indexnow_*` команды, checks, `django.tasks`); SQLAlchemy (события сессии, staging по кадрам транзакций, async); FastAPI и Flask (тонкие: ключ-файл, коллектор после ответа, роутер, CLI); Wagtail (свой пакет, сигналы публикации) |
| 26-python-wave-p.md | Волна P: таблица аудита «PHP-возможность → как есть / иначе / нет», порядок и оценка, версии (Python ≥ 3.11), гейт, инфраструктура (uv workspace `python/`, репо `indexnowkit/python` без сплитов, trusted publishing, docs под `/python/`), риски, `[решение]` ×13 |
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
