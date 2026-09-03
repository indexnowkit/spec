# 10. PHP: `indexnowkit/core`

PHP ≥ 8.2. Зависимости: `psr/http-client`, `psr/http-factory`, `psr/http-message`, `psr/log`,
`psr/simple-cache` (интерфейсы), `php-http/discovery` (обнаружение клиента). Без Guzzle/Symfony
HttpClient в require; в `suggest`. Тулинг: PHPUnit 11, phpstan level 9, php-cs-fixer (PER-CS 2),
rector для матрицы версий. Монорепо `indexnowkit/php` с subtree split по пакетам.

## Пространство имён

`IndexNowKit\` : `Client`, `Config`, `Engine` (enum), `Event` (enum `Created|Updated|Deleted`), `Result`, `ResultStatus`,
`Submitter`, `SubmitterInterface`, `IndexNow` (фасад),
`Key\KeyProviderInterface|StaticKeyProvider|KeyGenerator|KeyValidator`,
`Url\UrlNormalizerInterface|UrlNormalizer|UrlResolverInterface|RouteUrlResolverInterface|ResolverLocatorInterface|AttributeUrlResolver|GuardedUrlResolver`,
`Attribute\IndexNow|AttributeReaderInterface|AttributeReader` (+ `@internal` `ParamExtractor`, `PublishGuard`),
`Collector\Collector`, `Debounce\DebounceStoreInterface|MemoryDebounceStore|Psr16DebounceStore|NullDebounceStore`,
`Throttle\ThrottleInterface|TokenBucket|NullThrottle`, `Retry\RetryPolicy|RetryingSubmitter`,
`Dispatch\DispatcherInterface|SyncDispatcher|CallableDispatcher|NullDispatcher`, `Http\TransportInterface|Psr18Transport`,
`Sitemap\SitemapReader`, `Check\Checker`, `Exception\IndexNowException|ConfigurationException|InvalidUrlException|InvalidArgumentException`.

Реализовано в 0.2.0 (2026-09-03). Отличия от первоначального наброска: троттл живёт в `Client` (один токен на HTTP-запрос),
`GuardedUrlResolver` — единственная точка «объект → URL» для фасада и ORM-хуков (никогда не бросает), `SitemapReader`
потоковый (XMLReader) с ограничением на host/глубину/размер, `Psr18Transport::discover(timeout:)` настраивает
symfony/http-client или Guzzle (таймаут, без редиректов).

## Публичный API

```php
$indexNow = IndexNowKit\IndexNow::create(Config::fromArray([...]));   // фабрика с discovery
$results = $indexNow->submit(['https://www.example.com/a', '/b']);       // Result[]
$indexNow->submitEntity($post, Event::Updated);                          // через UrlResolver
```

`IndexNow` фасад = Submitter + Collector + Dispatcher, для использования без фреймворка.

## Атрибут (общий для всех PHP-адаптеров)

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
final readonly class IndexNow {
    public function __construct(
        public ?string $route = null,          // имя маршрута фреймворка
        public array $params = [],             // routeParam => property|getter|'expr'
        public ?string $resolver = null,       // FQCN UrlResolverInterface
        public ?string $when = null,           // имя метода/свойства bool
        public array $events = ['created','updated','deleted'],
        public array $fields = [],             // пусто = любые
    ) {}
}
```

Чтение атрибута: `ReflectionClass::getAttributes(IndexNow::class)`, результат кешируется
адаптером (Doctrine `loadClassMetadata`, Laravel static array). Core даёт
`Attribute\AttributeReader::read(class-string): ?IndexNow`.

`params` значения: имя свойства/геттера (`slug` → `$obj->slug` или `getSlug()`), либо
property-path через `symfony/property-access`, если установлен (в `suggest`).

`UrlResolverInterface::resolve(object $subject, Event $event): iterable<string>`; адаптеры
реализуют `RouteUrlResolver` на своём роутере. Возвращаемые URL могут быть относительными,
core дополняет `base_url`.

## Транспорт

`Psr18Transport(ClientInterface, RequestFactoryInterface, StreamFactoryInterface)`.
`IndexNow::create()` использует `Psr18ClientDiscovery`/`Psr17FactoryDiscovery`. Timeout
задаётся клиентом пользователя; core документирует рекомендованные 10 s / 3 s (sync).

## Debounce

`Psr16DebounceStore(CacheInterface $cache, string $prefix = 'indexnow_')`: ключ `sha1(url)`,
TTL = `debounce.per_url`. Адаптеры подставляют `cache.app` (Symfony) / `Cache::store()` (Laravel).

## CLI

`vendor/bin/indexnow key|check|submit|sitemap` (symfony/console в `require-dev`, бинарник
работает если console установлен; адаптеры дают свои команды). Конфиг из env `INDEXNOW_*`.

## Conformance

`tests/Conformance/CoreConformanceTest.php`, C01–C22 (C13 через `RetryingSubmitter`). YAML-фикстуры из репо `spec` — позже,
когда появится второй язык.
