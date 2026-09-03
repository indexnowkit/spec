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
final class IndexNowDriverMiddleware implements Doctrine\DBAL\Driver\Middleware {
    public function wrap(Driver $driver): Driver { return new IndexNowDriver($driver, $this->staging); }
}
final class IndexNowConnection extends AbstractConnectionMiddleware {
    public function commit(): void   { parent::commit();   $this->staging->commit($this); }
    public function rollBack(): void { parent::rollBack(); $this->staging->discard($this); }
}
```

`TransactionStaging` хранит URL по объекту соединения (`SplObjectStorage`): `stage(conn, urls)`,
`commit(conn)` → `Collector::add`, `discard(conn)`. Autocommit (нет `beginTransaction`) для
`flush()` невозможен: ORM всегда оборачивает flush в транзакцию, значит commit гарантирован.

Fallback без middleware (если пользователь не может зарегистрировать middleware): опция
`commit_detection: post_flush`, документированная как небезопасная при внешних транзакциях.

## ORM listener

`IndexNowListener` подписан на `loadClassMetadata`, `onFlush`, `postFlush`.

- `loadClassMetadata`: читает `#[IndexNow]` через `ClassMetadata::getReflectionClass()`,
  кладёт в собственный кеш (массив на процесс + опционально PSR-6, ключ по классу). Иерархия:
  атрибут ищется по родительским классам (`getParentClass()`), первый найденный.
- `onFlush`: обход `UnitOfWork::getScheduledEntityInsertions()`, `...Updates()`,
  `...Deletions()`. Для updates фильтр `fields` по `UnitOfWork::getEntityChangeSet($entity)`
  (ключи). Для deletions вычислить URL **здесь** (данные живы). Для insertions ID ещё нет,
  поэтому вычислять URL нельзя: запоминаем сам объект в `pendingInserts`.
- `postFlush`: для `pendingInserts` теперь есть ID, резолвим URL. Собранные URL всех трёх
  типов передаются в `TransactionStaging::stage($em->getConnection()->getNativeConnection()`-independent
  handle — используем `$em->getConnection()` как ключ, middleware маппит внутреннее соединение
  на внешнее через `ConnectionRegistry`, регистрируемый бандлом; standalone передаёт явно).
- `when`: вызывается в `onFlush`/`postFlush`; переход `true → false` определяется по changeset
  поля из `when`, если оно скалярное свойство и входит в changeset: шлём как `Deleted`.
- Переход `false → true` = `Created`.

Регистрация standalone:

```php
$em->getEventManager()->addEventSubscriber(new IndexNowListener($indexNow, $reader));
$config->setMiddlewares([...$config->getMiddlewares(), new IndexNowDriverMiddleware($staging)]);
```

## Ограничения (A13)

DQL/QueryBuilder `UPDATE`/`DELETE`, `Connection::executeStatement` события не вызывают.
`IndexNow::submit()` вручную. `postFlush` для сущностей, вставленных через `INSERT ... SELECT`, не применим.

## Гедмо-совместимость

Слушатель совместим с Gedmo (Sluggable вычисляет slug в `onFlush` раньше нас при условии
приоритета; в бандле наш listener регистрируется с `priority: -100`, чтобы сработать после Gedmo).

## Тесты

sqlite pdo, ORM 2.19/3.x × DBAL 3/4. A05: `wrapInTransaction` с исключением внутри после
`flush()`. A06: три persist один flush → один POST.
