# 32. JS/TS: `@indexnowkit/typeorm` (TypeORM 0.3.x)

## Установка

```ts
import { IndexNowSubscriber, IndexNowUrl } from '@indexnowkit/typeorm';
@Entity() @IndexNowUrl<Post>({ url: p => `/posts/${p.slug}`, when: p => p.published, fields: ['slug','title'] })
class Post {}
new DataSource({ ..., subscribers: [IndexNowSubscriber.for(indexNow)] });   // или indexNow.model('Post', ...) без декоратора
```

Декоратор пишет в `Reflect`-независимый `WeakMap<Function, ModelOptions>` (без `reflect-metadata`).

## Commit-safety

- `afterInsert`, `afterUpdate` (`event.updatedColumns` → `fields`), `beforeRemove` (URL до
  удаления, `event.entity`; `afterRemove` теряет id), `afterSoftRemove`, `afterRecover`.
  Все срабатывают внутри открытой транзакции (typeorm#2816).
- Staging по `event.queryRunner`: `WeakMap<QueryRunner, string[]>`. `afterTransactionCommit(event)`
  → flush для `event.queryRunner`; `afterTransactionRollback` → discard.
- Если `queryRunner.isTransactionActive === false` в момент `afterInsert` (autocommit, не в
  `dataSource.transaction()`), URL идёт сразу в Collector.
- Вложенные транзакции: TypeORM использует savepoints и `afterTransactionCommit` вызывается
  только на реальном commit (проверить при реализации; помечено как неподтверждённое в
  исследовании). Тест A05 обязателен.
- `createQueryBuilder().update()/delete()`, `repository.update()` (по критериям) не грузят
  сущность: `afterUpdate` вызывается с `event.entity` undefined/partial. Документировано A13
  для partial; при наличии `databaseEntity` резолвим из неё.

## NestJS

`@indexnowkit/nestjs` (37) регистрирует subscriber через `@nestjs/typeorm` `TypeOrmModule.forRoot({ subscribers })`
или `dataSource.subscribers.push` в `onModuleInit`.
