# 25. Python: Wagtail

## Решение

Существует зрелый `wagtail-indexnow` (RealOrangeOne, 0.2.0, 2025-08, Wagtail 6–7, дебаунс
10 мин). Не конкурируем. План:

1. PR в `wagtail-indexnow`: перевести HTTP-слой на `indexnowkit` core (дебаунс-store,
   батчинг, обработка 202/429, `check`). Сохранить их публичный API.
2. Если мейнтейнер не отвечает 30 дней: пакет `indexnowkit-wagtail` как тонкий адаптер
   (ниже), с явной ссылкой на оригинал в README.

## Адаптер (если понадобится)

- Сигналы `wagtail.signals.page_published`, `page_unpublished` (глобально, все Page).
  URL: `page.full_url` (Wagtail Site даёт host сам; `base_url` не нужен, но `hosts` карта
  ключей для мультисайта нужна).
- `page_published` уже после commit в контексте Wagtail revision publish; тем не менее
  оборачиваем в `transaction.on_commit`.
- Остальное (settings, key-роут, команды, checks) из 21 как зависимость `indexnowkit-django`.
- Не нужен миксин: все страницы публикуемы; исключения через `INDEXNOW["WAGTAIL_EXCLUDE"] = ["app.ModelName"]`.
