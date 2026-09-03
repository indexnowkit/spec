# 11. PHP: `indexnowkit/doctrine` (Doctrine ORM 2.19+/3.x, DBAL 3.x/4.x)

Библиотека слушателей, используется напрямую (standalone) и Symfony-бандлом (12).

## Проблема

В ORM нет события «после commit» (doctrine/orm #6825, #7103, #6292 отклонены). `postFlush`
срабатывает до внешнего `COMMIT`, если `flush()` обёрнут в `wrapInTransaction()` или ручную
транзакцию. DBAL `Events::onTransactionCommit` удалён в DBAL 4 вместе с EventManager.

## Решение: DBAL driver middleware

DBAL `Connection::commit()` вызывает драйверный `commit()` только при nesting level 0
(вложенные уровни это savepoint через SQL). Поэтому middleware уровня драйвера видит ровно
реальные commit/rollback в DBAL 3 и 4 одинаково.

```php
final class IndexNowMiddleware implements Doctrine\DBAL\Driver\Middleware {
    public function wrap(Driver $driver): Driver { return new IndexNowDriver($driver, $this->staging); }
}
final class IndexNowConnection extends AbstractConnectionMiddleware {   // DBAL 4, void
    public function commit(): void   { parent::commit();   $this->staging->commit($this->native()); }
    public function rollBack(): void { parent::rollBack(); $this->staging->discard($this->native()); }
}
```

`IndexNowDriver` выбирает `IndexNowConnection` (DBAL 4, `void`) или `IndexNowConnectionV3` (DBAL 3, `bool`)
по сигнатуре `AbstractConnectionMiddleware::commit()`. `commit()`, бросивший исключение, тоже приводит к
`discard()` до проброса, иначе переиспользованное соединение отправит URL позже.

`TransactionStaging` живёт в **core** (`IndexNowKit\Transaction\TransactionStaging`), а не в этом пакете:
он одинаково нужен любому адаптеру, обещающему commit-safety. Хранит URL в `WeakMap` по объекту-scope
(нативное соединение): `stage(scope, urls)`, `commit(scope)` → sink, `discard(scope)`. Autocommit для
`flush()` невозможен: ORM всегда оборачивает flush в транзакцию.

## ORM listener

`IndexNowListener` подписан на `onFlush` и `postFlush` (`IndexNowListener::EVENTS`). Метаданные атрибутов
компилирует и кеширует `AttributeReader` из core, поэтому `loadClassMetadata` не нужен.

Вся классификация делегирована в core (`Url\ObjectChangeHandler` + `Attribute\ChangeClassifier`) и идёт
**по каждому правилу** класса:

- `getScheduledEntityInsertions()` → `createdEvents()`, резолв откладывается до `postFlush` (нет ID).
- `getScheduledEntityUpdates()` → `updatedEvents($entity, array_keys($changeSet), $changeSet)`. Правило,
  ставшее невидимым (`when: true → false`), возвращается как `Deleted` и резолвится **сразу в `onFlush`**,
  пока живо старое состояние; остальные откладываются. Одна сущность может дать обновление одной страницы
  и удаление другой в одном flush.
- `getScheduledCollectionUpdates()` + `getScheduledCollectionDeletions()`: изменённая to-many связь не
  входит в change set владельца, поэтому владелец переклассифицируется с именем поля связи как изменённым.
  Имя поля читается из `PersistentCollection::getMapping()` (ORM 2 — массив, ORM 3 — объект).
- `getScheduledEntityDeletions()` → `deletedEvents()`, резолв в `onFlush`. Правило, которое не применяется
  (черновик), не отправляется.

В `postFlush` отложенные правила резолвятся, каждый URL логируется на `debug` с происхождением
(`{class}#{rule} ({event}) -> {url}`), затем передача: при открытой транзакции — в `TransactionStaging` по
нативному соединению, иначе сразу в `Collector`. Если драйвер не отдаёт нативное соединение — `warning` и
отправка внутри транзакции (лучше, чем потеря).

Ничто из этого не бросает в приложение: некорректный атрибут, нечитаемый `when` или падающий резолвер
пишутся в лог через `GuardedUrlResolver`.

## Регистрация standalone

```php
$wiring = new IndexNowDoctrine($indexNow, $resolver, $logger, autoFlush: true);
$wiring->registerMiddleware($ormConfiguration);   // ДО DriverManager::getConnection()
$wiring->registerListener($entityManager);
```

`IndexNowDoctrine` собирает staging, listener и middleware и связывает sink со `IndexNowKit::collect()`;
`autoFlush` дополнительно вызывает `flush()` сразу после передачи (сценарий CLI-скрипта). Поля
`$wiring->staging|listener|middleware` доступны для регистрации в контейнере по отдельности.

`route:` требует моста `RouteUrlResolverInterface`; у standalone Doctrine его нет, поэтому доступны
`url:`, `urls:`, `resolver:` либо собственная реализация моста.

## Ограничения (A13)

DQL/QueryBuilder `UPDATE`/`DELETE`, `Connection::executeStatement` событий не вызывают.
`IndexNowKit::submit()` вручную. `postFlush` для сущностей, вставленных через `INSERT ... SELECT`, не применим.
Атрибуты не читаются с интерфейсов и трейтов (PHP не наследует их, маппинг Doctrine ведёт себя так же).

## Gedmo-совместимость

Слушатель должен работать после Gedmo Sluggable, который пишет slug в `onFlush`; в бандле listener
регистрируется с `priority: -100`.

## Тесты

sqlite pdo, ORM 2.19/3.x × DBAL 3/4. A05: `wrapInTransaction` с исключением после `flush()`. A06: три
persist один flush → один POST. A15–A20 (см. 03): несколько правил, удаление по правилу, `when`-геттер,
удаление черновика, `via`, изменение коллекции.
