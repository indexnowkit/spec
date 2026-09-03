# 37. JS/TS: `@indexnowkit/nestjs` (NestJS 10/11)

## Установка

```ts
@Module({ imports: [IndexNowModule.forRoot({ key: process.env.INDEXNOW_KEY, baseUrl: '...', dispatch: 'queue' })] })
// forRootAsync({ useFactory, inject: [ConfigService] })
```

`ConfigurableModuleBuilder` → `forRoot/forRootAsync`, `MODULE_OPTIONS_TOKEN`. Экспортирует
`IndexNowService` (обёртка core: `submit`, `submitRecord`), `IndexNowKeyController`
(`@Controller() @Get(':key.txt')`, отключается опцией), `@IndexNowUrl()` декоратор из 32.

## Интеграции

- TypeORM: `IndexNowModule.forFeature({ typeorm: true })` регистрирует subscriber через
  `DataSource` из `@nestjs/typeorm` в `onModuleInit`.
- Prisma: пользователь применяет `$extends(withIndexNow(...))` в своём `PrismaService`;
  модуль даёт `IndexNowService.prismaExtension(models)`.
- Mongoose: `MongooseModule.forFeature` со `schema.plugin(...)`.
- Dispatch `queue`: если установлен `@nestjs/bullmq`, опция `bullmq: { queue: 'indexnow' }`
  регистрирует `Processor` с backoff по 01; иначе in-process `deferredDispatcher(setImmediate)`.
- Collector: interceptor `IndexNowCollectorInterceptor` (глобальный, `APP_INTERCEPTOR`)
  открывает ALS-область на запрос и flush в `finalize`.
- Health: `@nestjs/terminus` indicator при наличии.
