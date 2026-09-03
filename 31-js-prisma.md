# 31. JS/TS: `@indexnowkit/prisma` (Prisma 6/7)

`$use` middleware удалён (6.14). Используем `$extends`. Peer: `@prisma/client`.

## Установка

```ts
import { withIndexNow } from '@indexnowkit/prisma';
const prisma = new PrismaClient().$extends(withIndexNow(indexNow, {
  models: { post: { url: p => `/posts/${p.slug}`, when: p => p.published, fields: ['slug','title','body','published'] } },
}));
```

Типы `models` выводятся из `Prisma.ModelName` и payload-типов, `url` получает typed record.

## Детекция изменений

Query extension `$allModels.$allOperations` перехватывает `create`, `update`, `upsert`,
`delete`, `createManyAndReturn`, `updateManyAndReturn` (7.x). `createMany`, `updateMany`,
`deleteMany`, `$executeRaw` не возвращают записи: документировано A13, кроме `...AndReturn`.
`fields`: для `update` сравнить `args.data` ключи с `fields` (без чтения старой записи;
опция `fetchPrevious: true` делает `findUnique` до update для точного `when`-перехода).
`delete`: URL по результату (`delete` возвращает удалённую запись) — данные есть.

## Commit-safety

Prisma не имеет after-commit хука. Внутри `$transaction(async tx => ...)` extension видит
`tx` как клиент с `$transaction`-контекстом. Реализация:

- Extension определяет, что операция выполняется внутри interactive transaction по
  внутреннему признаку `(this as any)[Symbol.for('prisma.client.transaction.id')]`
  (нестабильно; проверить по версии) **либо** через обёртку: пакет экспортирует
  `indexNowTransaction(prisma, fn)`, которая создаёт ALS-контекст «в транзакции», внутри
  которого extension складывает URL в staging, а после успешного резолва промиса
  `$transaction` делает flush; при reject отбрасывает.
- Без обёртки (обычный `prisma.$transaction`): URL складываются в staging ALS-контекста, если
  адаптер не может определить commit, отправка происходит немедленно после операции,
  что документировано как «unsafe within $transaction, use indexNowTransaction». Батч
  (`prisma.$transaction([...])` массив) обрабатывается той же обёрткой.
- Вне транзакций: operation resolved = commit. URL → `Collector` (ALS области запроса) или
  напрямую dispatcher.

Открытый вопрос (проверить при реализации): наличие публичного способа узнать
transaction-контекст в `$extends`; если появится `$on('commit')` в Prisma 7, перейти на него.

## Тесты

sqlite Prisma schema, vitest, A01–A14 (A02/A05 через `indexNowTransaction` + throw).
