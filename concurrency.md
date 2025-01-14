---
git: aac0a3fa319d0ac84d546957c4ca30ca0ec4c2df
---

# Параллелизм

<a name="introduction"></a>
## Введение

> [!WARNING]
> Фасад Laravel `Concurrency` в настоящее время находится в стадии бета-тестирования, пока мы собираем отзывы сообщества.

Иногда вам может потребоваться выполнить несколько медленных задач, не зависящих друг от друга. Во многих случаях существенного повышения производительности можно добиться, выполняя задачи одновременно. Фасад Laravel `Concurrency` предоставляет простой и удобный API для одновременного выполнения замыканий.

<a name="concurrency-compatibility"></a>
#### Совместимость параллелизма

Если вы обновились до Laravel 11.x из приложения Laravel 10.x, вам может потребоваться добавить `ConcurrencyServiceProvider` в массив `providers` в файле конфигурации `config/app.php` вашего приложения:

```php
'providers' => ServiceProvider::defaultProviders()->merge([
    /*
     * Поставщики пакетных услуг...
     */
    Illuminate\Concurrency\ConcurrencyServiceProvider::class, // [tl! add]

    /*
     * Поставщики услуг приложений...
     */
    App\Providers\AppServiceProvider::class,
    App\Providers\AuthServiceProvider::class,
    // App\Providers\BroadcastServiceProvider::class,
    App\Providers\EventServiceProvider::class,
    App\Providers\RouteServiceProvider::class,
])->toArray(),
```

<a name="how-it-works"></a>
#### Как это работает

Laravel обеспечивает параллелизм путем сериализации заданных замыканий и отправки их скрытой команде Artisan CLI, которая десериализует замыкания и вызывает их в собственном PHP-процессе. После вызова замыкания полученное значение сериализуется обратно в родительский процесс.

Фасад `Concurrency` поддерживает три драйвера: `process` (по умолчанию), `fork` и `sync`.

Драйвер `fork` обеспечивает улучшенную производительность по сравнению с драйвером по умолчанию `process`, но его можно использовать только в контексте CLI PHP, поскольку PHP не поддерживает разветвление во время веб-запросов. Перед использованием драйвера `fork` вам необходимо установить пакет `spatie/fork`:

```bash
composer require spatie/fork
```

Драйвер `sync` в первую очередь полезен во время тестирования, когда вы хотите отключить весь параллелизм и просто последовательно выполнить заданные замыкания внутри родительского процесса.

<a name="running-concurrent-tasks"></a>
## Запуск параллельных задач

Для запуска параллельных задач вы можете вызвать метод `run` фасада `Concurrency`. Метод `run` принимает массив замыканий, которые должны выполняться одновременно в дочерних процессах PHP:

```php
use Illuminate\Support\Facades\Concurrency;
use Illuminate\Support\Facades\DB;

[$userCount, $orderCount] = Concurrency::run([
    fn () => DB::table('users')->count(),
    fn () => DB::table('orders')->count(),
]);
```

Чтобы использовать конкретный драйвер, вы можете использовать метод `driver`:

```php
$results = Concurrency::driver('fork')->run(...);
```

Или, чтобы изменить драйвер параллелизма по умолчанию, вам следует опубликовать файл конфигурации `concurrency` с помощью Artisan-команды `config:publish` и обновить параметр `default` в файле:

```bash
php artisan config:publish concurrency
```

<a name="deferring-concurrent-tasks"></a>
## Отсрочка параллельных задач

Если вы хотите одновременно выполнить массив замыканий, но вас не интересуют результаты, возвращаемые этими замыканиями, вам следует рассмотреть возможность использования метода `defer`. Когда вызывается метод `defer`, данные замыкания не выполняются немедленно. Вместо этого Laravel будет выполнять замыкания одновременно после отправки пользователю HTTP-ответа:

```php
use App\Services\Metrics;
use Illuminate\Support\Facades\Concurrency;

Concurrency::defer([
    fn () => Metrics::report('users'),
    fn () => Metrics::report('orders'),
]);
```
