# 42. Java/Kotlin: `dev.indexnowkit:indexnow-core`, `indexnow-spring-boot-starter`

Java ≥ 17, Spring Boot ≥ 3.2 (RestClient). Gradle multi-module: `core`, `spring-boot`
(autoconfigure), `spring-boot-starter` (POM). Не префиксовать `spring-boot-*` (зарезервировано).
Публикация в Maven Central через Central Publishing Portal plugin (OSSRH закрыт 2025-06).

## Core

- Zero-dep: `java.net.http.HttpClient`. Jackson не нужен: JSON body собирается вручную
  (структура плоская), экранирование по RFC 8259 в `JsonWriter` (тест на `"` и `\`).
- `IndexNowClient`, `IndexNowConfig` (builder), `Submitter`, `DebounceStore` (`InMemoryDebounceStore`,
  `Caffeine` не тянем), `KeyGenerator`, `Result` record, `Engine` enum.
- Логирование: `System.Logger` в core, SLF4J-мост в spring-модуле.

## Объявление сущности

```java
@Entity
public class Post implements IndexNowIndexable {
    @Override public List<String> indexNowUrls() { return List.of("/posts/" + slug); }
    @Override public boolean indexNowPublished() { return published; }
}
```

Альтернатива без изменения сущности: bean `IndexNowUrlResolver<Post>` (generic интерфейс,
starter собирает все резолверы по типу).

## Commit-safety (spring-boot модуль)

Двухшаговый паттерн:

1. Глобальный JPA listener через Hibernate `Integrator` (`PostInsertEventListener`,
   `PostUpdateEventListener`, `PreDeleteEventListener`/`PostDeleteEventListener`) вместо
   `@EntityListeners` на каждой сущности (пользователь не должен трогать сущности). Для
   `PostUpdate` фильтр по `dirtyProperties` и `indexnow.fields`. Listener публикует
   `IndexNowUrlsCollectedEvent(urls)` через `ApplicationEventPublisher`.
2. `@TransactionalEventListener(phase = AFTER_COMMIT)` в `IndexNowDispatchListener`
   передаёт в `Dispatcher`. Без активной транзакции `fallbackExecution = true` шлёт сразу.
   JPA-события во время flush до commit, поэтому шаг 2 обязателен.
- Delete: URL считается в `PreDelete`.
- Bulk JPQL/Criteria `update/delete` события не вызывают (A13).
- Батчинг на транзакцию: события накапливаются в `TransactionSynchronizationManager.bindResource`,
  один `afterCommit` с объединённым списком.

## Dispatch

- `SyncDispatcher` (после commit, в потоке запроса) и `AsyncDispatcher` (`@Async` на
  выделенном `ThreadPoolTaskExecutor` bean `indexNowExecutor`, bounded queue 1000,
  `CallerRunsPolicy`). Дефолт: async, если есть `TaskExecutor`; иначе sync.
- Retry: `spring-retry` не тянем; собственный backoff в `AsyncDispatcher` по 01.
- Опционально Spring Batch/Kafka не поддерживаем; пользователь может подменить `Dispatcher` bean.

## Autoconfiguration

- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
- `@ConfigurationProperties("indexnow")` с полями из 02 (`indexnow.key`, `indexnow.base-url`,
  `indexnow.engines`, `indexnow.dispatch`, `indexnow.debounce.per-url`...). Metadata JSON
  для IDE-автодополнения.
- `@ConditionalOnProperty("indexnow.key")` для всего; `@ConditionalOnClass(EntityManager)`
  для JPA-части; `@ConditionalOnWebApplication` для контроллера ключа
  `GET /{key}.txt` (`@RestController`, 404 для чужого).
- Actuator: `HealthIndicator` `indexnow` (последний результат, счётчик 403 подряд) при
  наличии actuator; Micrometer counters по 02 при наличии `MeterRegistry`.
- Кэш дебаунса: если есть `CacheManager`, `CacheDebounceStore("indexnow")`, иначе in-memory.

## CLI

Spring Shell не тянем. `indexnow-cli` fat-jar в core-модуле: `java -jar indexnow-cli.jar key|check|sitemap`.
Плюс `ApplicationRunner` при `indexnow.check-on-startup=true` (по умолчанию `true` в
non-prod профилях) логирует результат `check`.

## Kotlin

Core совместим. Ktor/Exposed: нет entity-хуков, только ручной `submit` в
`transaction {}` после успешного завершения; отдельный адаптер не планируем.

## Тесты

JUnit 5, `@SpringBootTest` с H2, WireMock (mock-сценарии), Testcontainers не обязателен.
Матрица Boot 3.2/3.4/3.5/4.0.

## Конкуренты

На Maven Central артефактов `indexnow` не найдено (перепроверить `search.maven.org` перед релизом).
