# 33. JS/TS: Drizzle, Mongoose, Sequelize

## `@indexnowkit/drizzle`

Drizzle без хуков (drizzle-orm#2266 открыт). Подход: явные обёртки, без магии.

```ts
import { indexNowDrizzle } from '@indexnowkit/drizzle';
const inx = indexNowDrizzle(indexNow, { tables: { [posts._.name]: { url: r => `/posts/${r.slug}`, when: r => r.published } } });
await inx.insert(db, posts).values(...)                           // returning() автоматически, submit после resolve
await inx.transaction(db, async (tx) => { await inx.update(tx, posts).set(...).where(...); });  // flush после commit
```

- `inx.insert/update/delete(dbOrTx, table)` возвращают Drizzle-билдеры с добавленным
  `.returning()` (pg/sqlite; для mysql `returning` нет: делаем `select` по PK после, опция
  `mysqlSelectAfter: true`) и `then`-хуком, который резолвит URL по вернувшимся строкам.
- В `inx.transaction` URL складываются в staging (ALS или объект `tx`), flush после успешного
  resolve, discard при throw. Вне транзакции: сразу.
- Обычные `db.insert()` без обёртки не отслеживаются: документировано, lint-правило не
  делаем.

## `@indexnowkit/mongoose`

- `schema.plugin(indexNowPlugin, { url, when, fields })`. Хуки `post('save')`, `post('findOneAndUpdate')`
  (с `new: true` для документа; иначе `findOne` после), `post(['deleteOne','findOneAndDelete'], {document: true, query: true})`,
  для `deleteOne` документ в `this`, URL до удаления через `pre`.
- Post-хуки срабатывают до `commitTransaction` (mongoose#8618). Если `this.$session()` есть:
  staging на `ClientSession` (`WeakMap`), flush в обёртке `withIndexNowTransaction(session, fn)`
  или пользователь вызывает `indexNow.flushSession(session)` после `commitTransaction()`.
  Без сессии: сразу.
- `updateMany`, `deleteMany` → A13.

## `@indexnowkit/sequelize`

- Единственный ORM с нативным `transaction.afterCommit`. `registerIndexNow(Model, { url, when, fields })`:
  хуки `afterCreate`, `afterUpdate` (`instance.changed()` для `fields`), `beforeDestroy` (URL) +
  `afterDestroy`, `afterBulkCreate`, `afterRestore`.
- Если `options.transaction`: `options.transaction.afterCommit(() => collector.add(urls))`;
  иначе сразу. CLS (`Sequelize.useCLS`) поддерживается автоматически (transaction в options).
- `Model.update()`/`destroy()` с `where` без `individualHooks: true` → A13.
