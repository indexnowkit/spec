# 40. Ruby: gem `indexnowkit` (+ Rails engine)

Один gem, core + опциональный Railtie/Engine (загружается только при `defined?(Rails)`).
Ruby ≥ 3.1, Rails ≥ 7.1 (после 7.1 порядок transactional callbacks детерминирован).

## Core

- `IndexNow::Client`, `IndexNow::Config`, `IndexNow::Submitter`, `IndexNow::Debounce::Memory|RailsCache`,
  `IndexNow::Key.generate`. HTTP: `Net::HTTP` из stdlib (без Faraday, нулевые зависимости);
  transport pluggable для webmock/VCR.
- `IndexNow.configure { |c| c.key = ENV["INDEXNOW_KEY"]; c.base_url = ...; c.dispatch = :active_job }`.

## Объявление модели

```ruby
class Post < ApplicationRecord
  index_now url: ->(post) { Rails.application.routes.url_helpers.post_url(post) },
            if: :published?, on: %i[create update destroy], fields: %w[slug title body status]
end
```

`index_now` макрос из `IndexNow::Model` (включается в `ActiveRecord::Base` через
`ActiveSupport.on_load(:active_record)`). Дефолт `url`: `url_helpers.polymorphic_url(record)`.
`default_url_options[:host]` берётся из `Rails.application.routes.default_url_options` либо
из `base_url` конфига (gem выставляет, если пусто).

## Commit-safety

- `after_create_commit`, `after_update_commit` (+ `saved_changes.keys & fields` для
  `on_fields`), `after_destroy_commit`. Единственная экосистема, где хук нативно после commit.
- Для destroy: URL считается в `before_destroy` и хранится в ivar, отправляется в
  `after_destroy_commit`.
- Переход `published → unpublished`: `saved_change_to_attribute?` по полю из `if`-условия
  детектится, если `fields` включает это поле; шлём как deleted.
- `update_all`, `insert_all`, `upsert_all`, `delete_all` колбэки не вызывают (A13). README: `IndexNow.submit(urls)`.
- Батчинг в запросе: `ActiveSupport::CurrentAttributes` (`IndexNow::Current.urls`), flush
  в `Rack` middleware после `@app.call` (после ответа не гарантирован в Puma; используем
  `dispatch: :active_job` по умолчанию в Rails, `:thread` вне Rails).

## Dispatch

- `IndexNow::SubmitJob < ActiveJob::Base`, `perform(urls)`, `retry_on IndexNow::RateLimited, wait: :polynomially_longer, attempts: 3`.
  Rails 8 Solid Queue работает без Redis, это аргумент в README.
- Дебаунс через `Rails.cache` (`IndexNow::Debounce::RailsCache`) по умолчанию в Rails.

## Engine

- `IndexNow::Engine` (isolate_namespace) с одним роутом `get "/:key.txt" => "keys#show"`,
  монтируется автоматически в корень через `Rails.application.routes.prepend`, если
  `serve_key_file` true. 404 для чужого ключа.
- Генератор `rails g index_now:install`: пишет `config/initializers/index_now.rb`, печатает
  ключ и подсказку про `.env`/credentials.
- Rake: `index_now:key`, `index_now:check`, `index_now:sitemap[url]`, `index_now:submit[url1,url2]`.

## Тесты

RSpec, sqlite, `webmock` на mock-сервер. Матрица Rails 7.1/7.2/8.0. Conformance A01–A14
(A05 через `transaction(requires_new: true)` + `raise ActiveRecord::Rollback`).

## Конкуренты

`rails-index-now` (0.1.4, 2025-07, ~11k загрузок): голый клиент + ActiveJob, без engine,
без дебаунса, без Yandex-конфига, без after_commit-макроса.
