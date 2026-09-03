# 43. .NET: `IndexNowKit`, `IndexNowKit.EntityFrameworkCore`, `IndexNowKit.AspNetCore`

Target `net8.0;net10.0`. Nullable enabled, `TreatWarningsAsErrors`. xUnit + FluentAssertions.

## Core `IndexNowKit`

- `IndexNowClient(HttpClient, IOptions<IndexNowOptions>, ILogger)`; JSON через `System.Text.Json`
  source-generated context (AOT-friendly). `IndexNowOptions` = поля из 02.
- `ISubmitter`, `IDebounceStore` (`MemoryDebounceStore` на `IMemoryCache`), `IDispatcher`,
  `IUrlResolver<T>`, `KeyGenerator`, `Result` record, `Engine` enum.
- DI: `services.AddIndexNow(configuration.GetSection("IndexNow"))` регистрирует typed
  `HttpClient` через `IHttpClientFactory` с `AddStandardResilienceHandler()`
  (Microsoft.Extensions.Http.Resilience) для 429/5xx.

## Объявление сущности

```csharp
public class Post : IIndexNowIndexable
{
    public IEnumerable<string> IndexNowUrls() => [$"/posts/{Slug}"];
    public bool IndexNowPublished => Published;
}
services.AddIndexNow(...).AddUrlResolver<Legacy>(l => [$"/legacy/{l.Id}"]);   // без интерфейса
```

## `IndexNowKit.EntityFrameworkCore` и commit-safety

- `IndexNowSaveChangesInterceptor : SaveChangesInterceptor`:
  - `SavingChangesAsync`: пройти `ChangeTracker.Entries()` со State `Added|Modified|Deleted`,
    для `Modified` сверить `Properties.Where(p => p.IsModified)` с `Fields`, вычислить URL,
    сохранить во внутренний список на `DbContext` (`ConditionalWeakTable<DbContext, List<string>>`).
  - `SavedChangesAsync`: если `context.Database.CurrentTransaction is null` → dispatch сразу.
    Иначе зарегистрировать flush на commit: EF Core не имеет события commit транзакции на
    `IDbContextTransaction`, поэтому используем `IDbTransactionInterceptor.TransactionCommittedAsync`
    (тот же класс реализует оба интерфейса) и `TransactionRolledBackAsync` (очистка).
    Это даёт A01–A06 без outbox.
  - `SaveChangesFailedAsync`: очистка.
- Регистрация: `options.UseIndexNow(serviceProvider)` расширение для `DbContextOptionsBuilder`
  или `services.AddDbContext<AppDb>((sp, o) => o.AddInterceptors(sp.GetRequiredService<IndexNowSaveChangesInterceptor>()))`.
- `ExecuteUpdate/ExecuteDelete` (EF 7+) интерсепторы SaveChanges не вызывают (A13).

## Dispatch

- `ChannelDispatcher`: `Channel<string>` bounded 10 000, `BackgroundService` `IndexNowDispatchService`
  батчит по `Batch.MaxUrls` или `FlushInterval` (2 s), graceful drain на `StopAsync`.
  Дефолт в ASP.NET Core. `SyncDispatcher` для консольных приложений.

## `IndexNowKit.AspNetCore`

- `app.MapIndexNowKeyFile()` → `MapGet("/{key}.txt")`, `Results.Text(key, "text/plain")` или 404.
- `IHealthCheck` `indexnow`.
- `IHostedService` регистрируется в `AddIndexNow` при наличии `IHostApplicationLifetime`.

## CLI

`dotnet tool install -g IndexNowKit.Cli`: `indexnow key|check|submit|sitemap`.

## Тесты

xUnit, `Microsoft.EntityFrameworkCore.Sqlite` in-memory, `WireMock.Net` или
`HttpMessageHandler` stub на mock-сценарии. Conformance полностью.

## Конкуренты

`CodeHelper.API.IndexNow` (1.0.0, 2022, ~850 загрузок, net6): `Submit`/`SubmitBulk` без EF/DI.
