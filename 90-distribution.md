# 90. Дистрибуция, README-шаблон, конкуренты

## Цель дистрибуции

Запрос «`<framework> indexnow`» в Google/Packagist/PyPI/npm/GitHub на первой странице
показывает наш пакет. Для этого: имя пакета = `<framework>` + `indexnow`, одинаковые
keywords, одинаковый README, взаимные ссылки между пакетами, регистрация в каталогах.

## README-шаблон (обязателен для каждого пакета)

```
# <Framework> IndexNow — indexnowkit/<name>

Одна строка о том, что делает. Бейджи: version, downloads, CI, conformance, license.

## Кто получит уведомление
Yandex, Bing (и DuckDuckGo через Bing), Naver, Seznam, Yep. Google: нет, протокол Google
не поддерживает (ссылка на FAQ). Честность здесь = доверие.

## Установка (3 команды максимум)
## Настройка (env INDEXNOW_KEY + 5 строк конфига)
## Объявление модели (1 атрибут/миксин)
## Проверка: `<cli> check`
## Как это работает (после commit, дебаунс, батчинг, очередь) — 6 буллетов
## Ручная отправка, sitemap, мультисайт
## Ограничения (bulk-операции не триггерят хуки; 10 минут между повторами)
## Другие фреймворки (таблица всех пакетов семейства со ссылками)
## Лицензия MIT
```

Русская версия README (`README.ru.md`) обязательна для PHP/Python/JS пакетов: Yandex
основной потребитель, русскоязычные разработчики основная аудитория запроса «яндекс
indexnow symfony».

## Метаданные пакетов

- Keywords везде: `indexnow`, `seo`, `search-engine`, `yandex`, `bing`, `<framework>`,
  `<orm>`, `sitemap`.
- Описание пакета начинается с имени фреймворка: «Symfony bundle for IndexNow: …».
- Homepage: `https://indexnowkit.dev/<lang>/<framework>` (docs-сайт, VitePress, один на
  всё семейство, с переключателем языка).
- GitHub topics на каждом репозитории: `indexnow`, `seo`, `<framework>`.

## Каталоги и площадки (чеклист на релиз)

| Экосистема | Где регистрироваться |
|---|---|
| PHP | Packagist (тип `symfony-bundle`), Symfony Flex contrib recipe (PR в symfony/recipes-contrib), symfony.com/bundles (автоматически из Packagist), Laravel News «packages», laravel-package-list awesome, Reddit r/PHP, r/laravel, Habr |
| Python | PyPI trove classifiers `Framework :: Django :: 5.x`, djangopackages.org (создать grid «SEO»), awesome-django PR, Django forum «Show & Tell», Wagtail packages, Habr |
| JS | npm keywords, Next.js/Nuxt/Payload/Strapi marketplace или plugin directory, awesome-nextjs, awesome-nuxt, nuxt modules registry (`nuxt/modules` PR), Payload plugins list, dev.to |
| Ruby | rubygems.org, ruby-toolbox, awesome-ruby, Rails discussion |
| Go | pkg.go.dev (автоматически), awesome-go PR (жёсткие требования: тесты, покрытие) |
| Java | Maven Central (Sonatype), spring.io «Community Projects», awesome-spring |
| .NET | NuGet, awesome-dotnet |
| Общее | indexnow.org FAQ/integrations (PR или письмо в Bing Webmaster team), Yandex Webmaster блог/форум, Wikipedia-статья IndexNow (раздел «Implementations», только если пакет заметен), Hacker News «Show HN» один раз на всё семейство |

## Конкуренты (снимок 2026-09-03)

PHP (Packagist, downloads total):
- `laravel-freelancer-nl/laravel-index-now` 19.9k, `ymigval/laravel-indexnow` 18.2k (34★),
  `jweiland/indexnow` TYPO3 7.8k, `nemorize/indexnow` 2.6k, `hakone/indexnow` PSR-17 0.5k,
  `slimad/indexnow` + `slimad/indexnow-laravel` 0.25k (есть queue), `nomadicsoft/laravel-indexnow`
  76 (единственный с post-commit и полной обработкой кодов, новый), `symkit/sitemap-bundle`
  (IndexNow как побочная фича), `linderp/sulu-index-now-bundle`. Symfony/Doctrine general
  bundle отсутствует. Ни один не делает атрибут + дебаунс + post-commit + 429 backoff + общий core вместе.
Python (PyPI): `wagtail-indexnow` (10-минутный дебаунс есть), `index-now-for-python`
  (sitemap submit), `indexnow` (занято, старое). Django/SQLAlchemy/FastAPI пусто.
JS (npm, загрузок/мес): `indexnow` 361 (2021, брошен), `astro-indexnow` 11k (build-time, живой),
  `indexnow-submitter` 2.5k, `nextjs-indexnow` 198, `next-indexnow` 59, `strapi-indexnow` 52,
  `@studio-labs-ai/indexnow` 19, `indexnow-notify` 12, `payload-cms-seo-plugin` 1.1k (IndexNow
  выключен по умолчанию). Nuxt: `@nuxtjs/seo` даёт только рецепт, не автоотправку (nuxt-seo#306).
  Prisma/TypeORM/Drizzle/Mongoose/Sequelize/SvelteKit/NestJS/Payload пусто.
Другие: Ruby `rails-index-now` (11k, голый клиент), Go `usk81/easyindex` (4 импортера), Java
  ничего, .NET `CodeHelper.API.IndexNow` (2022, 850). Транзакционно-безопасных ORM-хуков нет ни у кого.

Наше отличие в одном абзаце README: «Отправка после commit, дебаунс 10 минут, батчи по
10 000, обработка 202/429, очередь фреймворка, один ключ на host из env, команда `check`,
единый conformance-suite на все языки».

## Лицензия и governance

- MIT везде.
- CONTRIBUTING одинаковый, conventional commits, релиз через GitHub Actions по тегу.
- SECURITY.md: единственные секреты это ключ; ключ не является секретом в строгом смысле
  (он публикуется в `/{key}.txt`), но злоумышленник с ключом может слать мусорные URL
  вашего host, поэтому не логировать ключ целиком (маскировать до 4 символов).
