# 44. Прочие экосистемы (без отдельных пакетов в фазах 1–4)

## Elixir / Phoenix / Ecto

Ecto принципиально без колбэков. Идиома: `Ecto.Multi` для БД, HTTP после
`Repo.transaction/1` вернул `{:ok, _}`. Библиотека `indexnow` на hex.pm: тонкий клиент
(`Req`), `IndexNow.submit/1`, `IndexNow.after_transaction/2` helper, Plug для `/{key}.txt`,
mix-задачи `indexnow.key|check`. Фаза 5, по спросу.

## Rust / Axum / SeaORM

`ActiveModelBehavior::after_save` внутри вызова, не после commit. Crate `indexnow`:
клиент на `reqwest`, `Debounce`, `axum` route для ключа. SeaORM-хук документируется как
«только вне транзакций», иначе ручной `submit` после `txn.commit()`. Фаза 5.

## Статические генераторы (Hugo, Jekyll, Eleventy, Gatsby, Astro, Docusaurus)

Нет хуков, паттерн: diff sitemap. Уже есть `bojieyang/indexnow-action` (GitHub Action,
v3) и `astro-indexnow`. Наш вклад: CLI `indexnowkit sitemap --diff <previous.xml|cache>`
в каждом core (JS-версия для npm-скриптов, Python для CI), опционально свой GitHub Action
`indexnowkit/action` как обёртка над JS CLI с кешем предыдущего sitemap через
`actions/cache`. Фаза 3 (JS CLI) без отдельного пакета.

## CMS на PHP (WordPress, Drupal, TYPO3, Magento, Craft, Statamic, Kirby)

Рынок занят плагинами (mihdan-index-now для WP, jweiland/indexnow для TYPO3 и т. д.).
Не идём. Исключение: Drupal-модуль на базе `indexnowkit/core` в фазе 5, если будет спрос.
