# 10. PHP: `indexnowkit/core`

PHP 8.2–8.5. Зависимости: `psr/http-client`, `psr/http-factory`, `psr/http-message`, `psr/log`, `psr/clock`,
`psr/event-dispatcher`, `psr/simple-cache` (интерфейсы), `php-http/discovery` (обнаружение клиента). Без
Guzzle/Symfony HttpClient в require; в `suggest`. Тулинг: PHPUnit 11, phpstan level 9, php-cs-fixer (PER-CS 2).
Монорепо `indexnowkit/php` с subtree split по пакетам.

## Пространство имён

`IndexNowKit\` : `IndexNowKit` (фасад), `Client`, `Config`, `Engine` (enum), `Event` (enum
`Created|Updated|Deleted`), `Result`, `ResultStatus`, `Reason` (enum), `Submitter`, `SubmitterInterface`,
`Version`,

- `Attribute\` : `IndexNow` (repeatable), `IndexNowDefaults`, `IndexNowUrl`, `UrlRule`, `RuleSet`, `RuleSource`,
  `RuleEvent`, `RuleCompiler`, `RuleRegistry`, `ChangeClassifier`, `ParamExtractor`, `AttributeReaderInterface`,
  `AttributeReader`, `Param\{ParamValue, Accessor, Value, Formatted, Call, Placeholder}`;
- `Url\` : `UrlNormalizerInterface`, `UrlNormalizer`, `Punycode`, `UrlResolverInterface`,
  `RouteUrlResolverInterface`, `ResolverLocatorInterface`, `ArrayResolverLocator`, `CallableUrlResolver`,
  `NullUrlResolver`, `AttributeUrlResolver`, `GuardedUrlResolver`, `ObjectChangeHandler`, `ResolvedUrl`;
- `Key\` : `KeyProviderInterface`, `StaticKeyProvider`, `KeyGenerator`, `KeyValidator`, `KeyFileResponder`;
- `Collector\` : `CollectorInterface`, `Collector`;
- `Debounce\` : `DebounceStoreInterface`, `MemoryDebounceStore`, `Psr16DebounceStore`, `NullDebounceStore`;
- `Throttle\` : `ThrottleInterface`, `TokenBucket`, `NullThrottle`; `Clock\SystemClock`;
- `Retry\` : `RetryPolicy`, `RetryingSubmitter`;
- `Dispatch\` : `DispatcherInterface`, `SyncDispatcher`, `CallableDispatcher`, `NullDispatcher`;
- `Http\` : `TransportInterface`, `Psr18Transport`, `LazyTransport`, `Response`, `Exception\TransportException`;
- `Transaction\TransactionStaging`, `Adapter\ConfigFactory` (спека 16),
  `Check\{Checker, CheckReport, CheckItem, CheckLevel}`,
  `Testing\{FakeTransport, ArrayLogger, FrozenClock, RecordingDispatcher}`,
  `Exception\{IndexNowException, ConfigurationException, InvalidUrlException, InvalidArgumentException}`.

Реализовано в 0.2.0. Отличия от первоначального наброска: троттл живёт в `Client` (один токен на HTTP-запрос),
`GuardedUrlResolver` — единственная точка «объект → URL» для фасада и ORM-хуков (никогда не бросает),
`ObjectChangeHandler` — общий блок ORM-хука, `SitemapReader` потоковый (документы в `Sitemap\Spool`: анонимный
temp-файл либо память на read-only FS, XMLReader читает через wrapper `indexnowkit-spool://`; gzip по кускам;
память не зависит от размера) с ограничением на host/глубину/размер, опцией `allowForeignHosts`, retry загрузки
документа (`fetchRetries`, сеть/5xx), локальными файлами и текстовыми sitemap; `SitemapSourceInterface` — контракт
источника для команд адаптеров, `IndexNowKit::sitemap()` / `$transport` в фасаде, `ObjectChangeHandler::renamed()`
(старые URL переименованной страницы как deleted), `Check\CheckInterface`/`CheckerInterface`,
`Testing\Conformance\CoreConformanceTestCase`, `engine_aliases`/`locale_hosts` в Config, `Http\StreamingTransportInterface::download($url, $sink)` — необязательное
расширение транспорта для чтения тела без буферизации (`Psr18Transport`, `LazyTransport`, `FakeTransport`), `Psr18Transport::discover(timeout:)` настраивает symfony/http-client или Guzzle (таймаут,
без редиректов), `LazyTransport` откладывает discovery до первого запроса. Debug-режим `transport.method: get`
из 01-protocol.md не реализован сознательно: POST покрывает все случаи.

Фасад назван `IndexNowKit`, а не `IndexNow`, чтобы не сталкиваться с атрибутом `Attribute\IndexNow`, который
пользователь пишет чаще.

## Публичный API

```php
$indexNow = IndexNowKit\IndexNowKit::create(Config::fromArray([...]));   // фабрика с discovery
$results = $indexNow->submit(['https://www.example.com/a', '/b']);        // Result[]
$indexNow->submitEntity($post, Event::Updated);                          // через UrlResolver
$indexNow->collect($urls); $indexNow->flush();                           // единица работы
$urls = $indexNow->urlsFor($post, Event::Deleted);                       // без отправки
$rows = $indexNow->explain($post, Event::Updated);                       // ResolvedUrl[]
$indexNow->resolver();  $indexNow->changes();                            // для ORM-хуков
```

Публичные свойства фасада: `config`, `submitter`, `collector`, `dispatcher`, `keys`, `attributes`.
Имена аргументов `create()` — часть обещания совместимости, порядок — нет: всегда именованные аргументы.

## Атрибуты (общие для всех PHP-адаптеров)

```php
#[Attribute(Attribute::TARGET_CLASS | Attribute::IS_REPEATABLE)]
final readonly class IndexNow {
    public function __construct(
        public ?string $route = null,                       // имя маршрута фреймворка
        public array $params = [],                          // routeParam => accessor|ParamValue
        public ?string $resolver = null,                    // id сервиса или FQCN UrlResolverInterface
        public ?string $via = null,                         // accessor к связанному объекту/коллекции
        public ?string $url = null,                         // accessor -> string|iterable<string>|null
        public array $urls = [],                            // литеральные URL
        public ?string $when = null,
        public array $whenFields = [],
        public ?array $fields = null,                       // null = наследовать, [] = любые
        ?array $events = null,
        public array|string|null $locales = null,
        public string|ParamValue|null $host = null,
        public ?string $name = null,
    ) {}
}

#[Attribute(Attribute::TARGET_CLASS)]  final readonly class IndexNowDefaults { /* when, whenFields, fields, events, locales */ }
#[Attribute(Attribute::TARGET_METHOD)] final readonly class IndexNowUrl      { /* when, whenFields, fields, events, name */ }
```

Ровно один источник на правило (`route|resolver|via|url|urls`), иначе `ConfigurationException` при компиляции.
Компиляция в `UrlRule[]`: `RuleCompiler::compile(ReflectionClass)` обходит иерархию от корня к листу, сливает
`IndexNowDefaults` по полям, правило с именем предка заменяет его. Кеш на класс — в `AttributeReader`.
Правила в рантайме — `RuleRegistry` (`register()` по классу, `registerFor()` по объекту).

Значения `params`: строка-accessor (свойство, геттер, `is`/`has`-метод, точечный путь, `self`) либо
`Param\{Accessor, Value, Formatted, Call}`; `Placeholder::Locale|Host` подставляются на каждый генерируемый URL.
`symfony/property-access` не используется.

`UrlResolverInterface::resolve(object $subject, Event $event): iterable<string>`.
`RouteUrlResolverInterface` — два метода: `locales(array|string): list<string|null>` и
`generate(string $route, array $params, ?string $locale, ?string $host): string`; реализуется адаптером.
Возвращаемые URL могут быть относительными, core дополняет их `base_url`.

## Транспорт

`Psr18Transport(ClientInterface, RequestFactoryInterface, StreamFactoryInterface)`;
`Psr18Transport::discover(?ClientInterface, ?float $timeout)` использует `Psr18ClientDiscovery`/
`Psr17FactoryDiscovery`. POST читает не более 2 КиБ тела (диагностика), GET — до 50 МиБ (максимум sitemap).
`LazyTransport` откладывает построение до первого запроса. `Response::parseRetryAfter()` — общий разбор
заголовка (delta-seconds и HTTP-date) с ограничением сверху.

## Debounce

`Psr16DebounceStore(CacheInterface $cache, string $prefix = 'indexnow_')`: ключ `prefix.sha1(url)`,
TTL = `debounce.per_url` (бандл Symfony передаёт префикс `indexnowkit_`). `MemoryDebounceStore` ограничен 50 000 записями. Отказ хранилища — fail-open:
отправка идёт без дедупликации, в лог уходит предупреждение.

## Тестовый инструментарий

`IndexNowKit\Testing` входит в публикуемый пакет (не `autoload-dev`): `FakeTransport` (записывает POST,
отдаёт ответы и исключения из очереди, `onGet()` для файлов ключа), `ArrayLogger`, `FrozenClock`,
`RecordingDispatcher`. На нём строятся тесты адаптеров и приложений.

## Документация

`README.md` / `README.ru.md` плюс `docs/`: `configuration.md`, `attribute-reference.md`,
`retries-and-queues.md`, `operations.md`, `testing.md`, `adapters.md` (гайд автора адаптера), `bc.md`
(поверхность публичного API и правила SemVer).

## Conformance

`tests/Conformance/CoreConformanceTest.php`, C01–C22 (C13 через `RetryingSubmitter`). YAML-фикстуры из репо
`spec` — позже, когда появится второй язык.

## Пакет `indexnowkit/sitemap` (core 0.4+)

Пространство `Sitemap\` (`SitemapReader`, `Spool`, `SpoolMode`, `SitemapEntry`, `SitemapSourceInterface`, `SitemapConfig`,
`Sitemap\Console\SitemapRunner`, `Sitemap\Check\SitemapSpoolCheck`) живёт в отдельном пакете-потребителе core; в core нет ни
кода, ни опций, ни текстов про sitemap (спека 16 §1). Адаптеры требуют пакет и строят reader через
`SitemapReader::fromConfig(SitemapConfig::fromArray($block), $transport, $logger)`. Набор для адаптеров core 0.4:
`Adapter\ConfigFactory`, фабрики `Http\TransportFactory`, `Debounce\DebounceStoreFactory`, `Dispatch\DispatcherFactory`,
`fromConfig()` у `Collector`, `TokenBucket`, `AttributeUrlResolver`, `KeyFileResponder`; `Console\ClassNameResolver`,
`Check\DebounceStoreCheck`, `Url\ArrayResolverLocator(locate:, hint:)`, `Url\RuleAwareUrlResolverInterface`.
