# 14. PHP: API Platform и CMS

## `indexnowkit/api-platform` (API Platform 3.4/4.x, Symfony-режим)

Тонкий пакет поверх 12. Задачи:

1. URL: IRI (`/api/posts/42`) не является публичной страницей. По умолчанию адаптер
   **ничего не шлёт**, пока в `#[IndexNow]` не указан `route`/`resolver` (публичный HTML URL).
   Опция `use_iri: true` для headless-случаев, где сам API отдаёт HTML (редко).
2. Триггер: не нужен отдельный, Doctrine listener из 11 срабатывает под `PersistProcessor`.
   Декоратор процессора не используем.
3. Ценность пакета: `IndexNowUrlResolver` с доступом к `IriConverterInterface` для тех, кто
   маппит IRI → фронтовой URL шаблоном `front_url_template: 'https://www.example.com/posts/{id}'`
   (плейсхолдеры из `uriVariables`).
4. Laravel-режим API Platform: работает через 13 без изменений.

Фаза 5, только при спросе.

## Sylius, Shopware, Drupal

- Sylius: Symfony+Doctrine, бандл 12 работает как есть. Рецепт в README: `when` на
  `isEnabled()`, локализованные `ProductTranslation` через кастомный резолвер (slug в
  translation), state-machine callback для транзитов.
- Shopware 6: DAL без Doctrine ORM. Плагин `indexnowkit/shopware` (фаза 5): подписка на
  `EntityWrittenContainerEvent` (после commit DAL), `SeoUrlPlaceholderHandlerInterface` для URL,
  `MessageQueue` Shopware для dispatch. Рынок: только платный store-плагин, открытый feature request.
- Drupal: `drupal.org/project/index_now` уже есть и полный. Не идём.
- WordPress: вне scope (см. 44).
