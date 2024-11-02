---
git: 690175ad7106a7b8ce6e6e8420872ca932622e84
---

# Контейнер служб (service container)

<a name="introduction"></a>
## Введение

Контейнер служб (service container, сервис-контейнер) Laravel – это мощный инструмент для управления зависимостями классов и выполнения внедрения зависимостей. Внедрение зависимостей – это причудливая фраза, которая по существу означает следующее: зависимости классов «вводятся» в класс через конструктор в виде аргументов или, в некоторых случаях, через методы-сеттеры. При создании класса или вызове методов фреймворк смотрит на список аргументов и, если нужно, создаёт экземпляры необходимых классов и сам подаёт их на вход конструктора или метода.

Давайте посмотрим на простой пример:

    <?php

    namespace App\Http\Controllers;

    use App\Services\AppleMusic;
    use Illuminate\View\View;

    class PodcastController extends Controller
    {
        /**
         * Создать новый экземпляр контроллера.
         *
         * @param  UserRepository  $users
         * @return void
         */
        public function __construct(
            protected AppleMusic $apple,
        ) {}

        /**
         * Показать информацию о данном подкасте.
         */
        public function show(string $id): View
        {
            return view('podcasts.show', [
                'podcast' => $this->apple->findPodcast($id)
            ]);
        }
    }

В этом примере `PodcastController` необходимо получить подкасты из источника данных, такого как Apple Music. Итак, мы **внедрим** сервис, способный извлекать подкасты. Поскольку служба внедрена, мы можем легко «имитировать» или создать фиктивную реализацию службы `AppleMusic` при тестировании нашего приложения.

Глубокое понимание контейнера служб Laravel необходимо для создания большого, мощного приложения, а также для внесения вклада в само ядро Laravel.

<a name="zero-configuration-resolution"></a>
### Неконфигурируемое внедрение

Если класс не имеет зависимостей или зависит только от других конкретных классов (не интерфейсов), контейнер не нужно инструктировать о том, как создавать этот класс. Например, вы можете поместить следующий код в свой файл `routes/web.php`:

    <?php

    class Service
    {
        // ...
    }

    Route::get('/', function (Service $service) {
        die($service::class);
    });

В этом примере, при посещении `/` вашего приложения, маршрут автоматически получит класс `Service` и внедрит его в обработчике вашего маршрута. Это меняет правила игры. Это означает, что вы можете разработать свое приложение и воспользоваться преимуществами внедрения зависимостей, не беспокоясь о раздутых файлах конфигурации.

К счастью, многие классы, которые вы будете писать при создании приложения Laravel, автоматически получают свои зависимости через контейнер, включая [контроллеры](/docs/{{version}}/controllers), [слушатели событий](/docs/{{version}}/events), [посредники](/docs/{{version}}/middleware ) и т.д. Кроме того, вы можете указать зависимости в методе `handle` обработки [заданий в очереди](/docs/{{version}}/queues). Как только вы почувствуете всю мощь автоматического неконфигурируемого внедрения зависимостей, вы почувствуете невозможность разработки без нее.

<a name="when-to-use-the-container"></a>
### Когда использовать контейнер

Благодаря неконфигурируемому внедрению, вы часто будете объявлять типы зависимостей в маршрутах, контроллерах, слушателях событий и других местах, не взаимодействуя с контейнером напрямую. Например, вы можете указать объект `Illuminate\Http\Request` в определении вашего маршрута, для того, чтобы легко получить доступ к текущему запросу. Несмотря на то, что нам никогда не нужно взаимодействовать с контейнером для написания этого кода, он управляет внедрением этих зависимостей за кулисами:

    use Illuminate\Http\Request;

    Route::get('/', function (Request $request) {
        // ...
    });

Во многих случаях, благодаря автоматическому внедрению зависимостей и [фасадам](/docs/{{version}}/facades), вы можете строить приложения Laravel без необходимости **когда-либо** вручную связывать или извлекать что-либо из контейнера. **В каких же случаях есть необходимость вручную взаимодействовать с контейнером?**. Давайте рассмотрим две ситуации.

Во-первых, если вы пишете класс, реализующий интерфейс, и хотите объявить тип этого интерфейса в конструкторе маршрута или класса, то вы должны [сообщить контейнеру, как получить этот интерфейс](#binding-interfaces-to-implementations). Во-вторых, если вы [пишете пакет Laravel](/docs/{{version}}/packages), которым планируете поделиться с другими разработчиками Laravel, вам может потребоваться связать службы вашего пакета в контейнере.

<a name="binding"></a>
## Связывание

<a name="binding-basics"></a>
### Основы связываний

<a name="simple-bindings"></a>
#### Простое связывание

Почти все ваши связывания в контейнере служб будут зарегистрированы в [поставщиках служб](/docs/{{version}}/providers), поэтому в большинстве этих примеров будет продемонстрировано использование контейнера в этом контексте.

Внутри поставщика служб у вас всегда есть доступ к контейнеру через свойство `$this->app`. Мы можем зарегистрировать связывание, используя метод `bind`, передав имя класса или интерфейса, которые мы хотим зарегистрировать, вместе с замыканием, возвращающим экземпляр класса:

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bind(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

Обратите внимание, что мы получаем сам контейнер в качестве аргумента. Затем мы можем использовать контейнер для извлечения под-зависимостей объекта, который мы создаем.

Как уже упоминалось, вы обычно будете взаимодействовать с контейнером внутри поставщиков служб; однако, если вы хотите взаимодействовать с контейнером в других частях приложения, вы можете сделать это через [фасад](/docs/{{version}}/facades) `App`:

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\Facades\App;

    App::bind(Transistor::class, function (Application $app) {
        // ...
    });


Вы можете использовать метод `bindIf` для регистрации привязки контейнера только в том случае, если привязка уже не была зарегистрирована для данного типа:

```php
$this->app->bindIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

> [!NOTE]  
> Нет необходимости привязывать классы в контейнере, если они не зависят от каких-либо интерфейсов. Контейнеру не нужно указывать, как создавать эти объекты, поскольку он может автоматически извлекать эти объекты с помощью рефлексии.

<a name="binding-a-singleton"></a>
#### Связывание одиночек

Метод `singleton` связывает в контейнере класс или интерфейс, который должен быть извлечен только один раз. При последующих обращениях к этому классу из контейнера будет возвращен полученный ранее экземпляр объекта:

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->singleton(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

Вы можете использовать метод `singletonIf` для регистрации синглтон-привязки контейнера только в том случае, если привязка уже не была зарегистрирована для данного типа:

```php
$this->app->singletonIf(Transistor::class, function (Application $app) {
    return new Transistor($app->make(PodcastParser::class));
});
```

<a name="binding-scoped"></a>
#### Связывание одиночек с заданной областью действия

Метод `scoped` связывает в контейнере класс или интерфейс, который должен быть извлечен только один раз в течение данного жизненного цикла запроса / задания Laravel. Хотя этот метод похож на метод `singleton` похож на метод `scoped`, экземпляры, зарегистрированные с помощью метода `scoped`, будут сбрасываться всякий раз, когда приложение Laravel запускает новый «жизненный цикл», например, когда [Laravel Octane](/docs/{{version}}/octane) обрабатывает новый запрос или когда [очереди](/docs/{{version}}/queues) обрабатывают новое задание:

    use App\Services\Transistor;
    use App\Services\PodcastParser;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->scoped(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

Вы можете использовать метод `scopedIf` для регистрации привязки контейнера с ограниченной областью действия, только если привязка еще не зарегистрирована для данного типа:

    $this->app->scopedIf(Transistor::class, function (Application $app) {
        return new Transistor($app->make(PodcastParser::class));
    });

<a name="binding-instances"></a>
#### Связывание экземпляров

Вы также можете привязать существующий экземпляр объекта в контейнере, используя метод `instance`. Переданный экземпляр всегда будет возвращен из контейнера при последующих вызовах:

    use App\Services\Transistor;
    use App\Services\PodcastParser;

    $service = new Transistor(new PodcastParser);

    $this->app->instance(Transistor::class, $service);

<a name="binding-interfaces-to-implementations"></a>
### Связывание интерфейсов и реализаций

Очень мощная функция контейнера служб – это его способность связывать интерфейс с конкретной реализацией. Например, предположим, что у нас есть интерфейс `EventPusher` и реализация `RedisEventPusher`. После того как мы написали нашу реализацию `RedisEventPusher` этого интерфейса, мы можем зарегистрировать его в контейнере следующим образом:

    use App\Contracts\EventPusher;
    use App\Services\RedisEventPusher;

    $this->app->bind(EventPusher::class, RedisEventPusher::class);

Эта запись сообщает контейнеру, что он должен внедрить `RedisEventPusher`, когда классу требуется реализация `EventPusher`. Теперь мы можем указать интерфейс `EventPusher` в конструкторе класса, который будет извлечен контейнером. Помните, что контроллеры, слушатели событий, посредники и некоторые другие типы классов в приложениях Laravel всегда выполняются с помощью контейнера:

    use App\Contracts\EventPusher;

    /**
     * Создать новый экземпляр класса.
     */
    public function __construct(
        protected EventPusher $pusher,
    ) {}

<a name="contextual-binding"></a>
### Контекстная привязка

Иногда у вас может быть два класса, которые используют один и тот же интерфейс, но вы хотите внедрить разные реализации в каждый класс. Например, два контроллера могут зависеть от разных реализаций [контракта](/docs/{{version}}/contracts) `Illuminate\Contracts\Filesystem\Filesystem`. Laravel предлагает простой и понятный интерфейс для определения этого поведения:

    use App\Http\Controllers\PhotoController;
    use App\Http\Controllers\UploadController;
    use App\Http\Controllers\VideoController;
    use Illuminate\Contracts\Filesystem\Filesystem;
    use Illuminate\Support\Facades\Storage;

    $this->app->when(PhotoController::class)
              ->needs(Filesystem::class)
              ->give(function () {
                  return Storage::disk('local');
              });

    $this->app->when([VideoController::class, UploadController::class])
              ->needs(Filesystem::class)
              ->give(function () {
                  return Storage::disk('s3');
              });

<a name="contextual-attributes"></a>
### Контекстуальные атрибуты

Поскольку контекстная привязка часто используется для внедрения реализаций драйверов или значений конфигурации, Laravel предлагает множество атрибутов контекстной привязки, которые позволяют внедрять эти типы значений без ручного определения контекстных привязок у ваших поставщиков услуг.

Например, атрибут `Storage` может использоваться для внедрения определенного [диска хранения](/docs/{{version}}/filesystem):

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Container\Attributes\Storage;
use Illuminate\Contracts\Filesystem\Filesystem;

class PhotoController extends Controller
{
    public function __construct(
        #[Storage('local')] protected Filesystem $filesystem
    )
    {
        // ...
    }
}
```

В дополнение к атрибуту `Storage`, Laravel предлагает атрибуты `Auth`, `Cache`, `Config`, `DB`, `Log`, `RouteParameter` и [`Tag`](#tagging):

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Container\Attributes\Auth;
use Illuminate\Container\Attributes\Cache;
use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\DB;
use Illuminate\Container\Attributes\Log;
use Illuminate\Container\Attributes\Tag;
use Illuminate\Contracts\Auth\Guard;
use Illuminate\Contracts\Cache\Repository;
use Illuminate\Contracts\Database\Connection;
use Psr\Log\LoggerInterface;

class PhotoController extends Controller
{
    public function __construct(
        #[Auth('web')] protected Guard $auth,
        #[Cache('redis')] protected Repository $cache,
        #[Config('app.timezone')] protected string $timezone,
        #[DB('mysql')] protected Connection $connection,
        #[Log('daily')] protected LoggerInterface $log,
        #[Tag('reports')] protected iterable $reports,
    )
    {
        // ...
    }
}
```

Кроме того, Laravel предоставляет атрибут `CurrentUser` для добавления текущего аутентифицированного пользователя в заданный маршрут или класс:

```php
use App\Models\User;
use Illuminate\Container\Attributes\CurrentUser;

Route::get('/user', function (#[CurrentUser] User $user) {
    return $user;
})->middleware('auth');
```

<a name="defining-custom-attributes"></a>
#### Определение пользовательских атрибутов

Вы можете создавать свои собственные контекстные атрибуты, реализуя контракт `Illuminate\Contracts\Container\ContextualAttribute`. Контейнер вызовет метод `resolve` вашего атрибута, который должен разрешить значение, которое должно быть введено в класс, использующий атрибут. В приведенном ниже примере мы повторно реализуем встроенный атрибут Laravel `Config`:

    <?php

    namespace App\Attributes;

    use Illuminate\Contracts\Container\ContextualAttribute;

    #[Attribute(Attribute::TARGET_PARAMETER)]
    class Config implements ContextualAttribute
    {
        /**
         * Create a new attribute instance.
         */
        public function __construct(public string $key, public mixed $default = null)
        {
        }

        /**
         * Resolve the configuration value.
         *
         * @param  self  $attribute
         * @param  \Illuminate\Contracts\Container\Container  $container
         * @return mixed
         */
        public static function resolve(self $attribute, Container $container)
        {
            return $container->make('config')->get($attribute->key, $attribute->default);
        }
    }

<a name="binding-primitives"></a>
### Связывание примитивов

Иногда у вас может быть класс, который получает некоторые внедренные классы, но также нуждается в примитиве, таком как целое число. Вы можете легко использовать контекстную привязку, чтобы внедрить любое значение, которое может понадобиться вашему классу:

    use App\Http\Controllers\UserController;

    $this->app->when(UserController::class)
              ->needs('$variableName')
              ->give($value);

Иногда класс может зависеть от массива экземпляров, объединенных [меткой](#tagging). Используя метод `giveTagged`, вы можете легко их внедрить:

    $this->app->when(ReportAggregator::class)
        ->needs('$reports')
        ->giveTagged('reports');

Если вам нужно внедрить значение из одного из конфигурационных файлов вашего приложения, то вы можете использовать метод `giveConfig`:

    $this->app->when(ReportAggregator::class)
        ->needs('$timezone')
        ->giveConfig('app.timezone');

<a name="binding-typed-variadics"></a>
### Связывание типизированных вариаций

Иногда у вас может быть класс, который получает массив типизированных объектов с использованием переменного количества аргументов (_прим. перев.: далее «вариации»_) конструктора:

    <?php

    use App\Models\Filter;
    use App\Services\Logger;

    class Firewall
    {
        /**
         * Создать новый экземпляр класса.
         */
        public function __construct(
            protected Logger $logger,
            Filter ...$filters,
        ) {
            $this->filters = $filters;
        }
    }

Используя контекстную привязку, вы можете внедрить такую зависимость, используя метод `give` с замыканием, которое возвращает массив внедряемых экземпляров `Filter`:

    $this->app->when(Firewall::class)
              ->needs(Filter::class)
              ->give(function (Application $app) {
                    return [
                        $app->make(NullFilter::class),
                        $app->make(ProfanityFilter::class),
                        $app->make(TooLongFilter::class),
                    ];
              });

Для удобства вы также можете просто передать массив имен классов, которые будут предоставлены контейнером всякий раз, когда для `Firewall` нужны экземпляры `Filter`:

    $this->app->when(Firewall::class)
              ->needs(Filter::class)
              ->give([
                  NullFilter::class,
                  ProfanityFilter::class,
                  TooLongFilter::class,
              ]);

<a name="variadic-tag-dependencies"></a>
#### Метки вариативных зависимостей

Иногда класс может иметь вариативную зависимость, указывающую на тип как переданный класс (`Report ...$reports`). Используя методы `needs` и `giveTagged`, вы можете легко внедрить все привязки контейнера с этой [меткой](#tagging) для указанной зависимости:

    $this->app->when(ReportAggregator::class)
        ->needs(Report::class)
        ->giveTagged('reports');

<a name="tagging"></a>
### Добавление меток

Иногда может потребоваться получить все привязки определенной «категории». Например, возможно, вы создаете анализатор отчетов, который получает массив из множества различных реализаций интерфейса `Report`. После регистрации реализаций `Report` вы можете назначить им метку с помощью метода `tag`:

    $this->app->bind(CpuReport::class, function () {
        // ...
    });

    $this->app->bind(MemoryReport::class, function () {
        // ...
    });

    $this->app->tag([CpuReport::class, MemoryReport::class], 'reports');

После того как службы помечены, вы можете легко все их получить с помощью метода `tagged`:

    $this->app->bind(ReportAnalyzer::class, function (Application $app) {
        return new ReportAnalyzer($app->tagged('reports'));
    });

<a name="extending-bindings"></a>
### Расширяемость связываний

Метод `extend` позволяет модифицировать извлеченные службы. Например, когда служба получена, вы можете выполнить дополнительный код для декорирования или конфигурирования службы. Метод `extend` принимает два аргумента: класс службы, который вы расширяете, и замыкание, которое должно возвращать модифицированную службу. Замыкание получает службу, которая извлечения, и экземпляр контейнера:

    $this->app->extend(Service::class, function (Service $service, Application $app) {
        return new DecoratedService($service);
    });

<a name="resolving"></a>
## Извлечение

<a name="the-make-method"></a>
### Метод `make`

Вы можете использовать метод `make` для извлечения экземпляра класса из контейнера. Метод `make` принимает имя класса или интерфейса, который вы хотите получить:

    use App\Services\Transistor;

    $transistor = $this->app->make(Transistor::class);

Если некоторые зависимости вашего класса не могут быть разрешены через контейнер, вы можете ввести их, передав их как ассоциативный массив в метод `makeWith`. Например, мы можем вручную передать конструктору аргумент `$id`, требуемый службой `Transistor`:

    use App\Services\Transistor;

    $transistor = $this->app->makeWith(Transistor::class, ['id' => 1]);

Метод `bound` может быть использован для определения, был ли класс или интерфейс явно привязан в контейнере:

    if ($this->app->bound(Transistor::class)) {
        // ...
    }

Если вы находитесь за пределами поставщика служб и не имеете доступа к переменной `$app`, вы можете использовать [фасад](/docs/{{version}}/facades) `App` для получения экземпляра класса из контейнера:

    use App\Services\Transistor;
    use Illuminate\Support\Facades\App;

    $transistor = App::make(Transistor::class);

Если вы хотите, чтобы сам экземпляр контейнера Laravel был внедрен в класс, извлекаемый контейнером, вы можете указать класс `Illuminate\Container\Container` в конструкторе вашего класса:

    use Illuminate\Container\Container;

    /**
     * Создать новый экземпляр класса.
     */
    public function __construct(
        protected Container $container,
    ) {}

<a name="automatic-injection"></a>
### Автоматическое внедрение зависимостей

Важно, что в качестве альтернативы, вы можете объявить тип зависимости в конструкторе класса, который извлекается контейнером, включая [контроллеры](/docs/{{version}}/controllers), [слушатели событий](/docs/{{version}}/events), [посредники](/docs/{{version}}/middleware ) и т.д. Кроме того, вы можете объявить зависимости в методе `handle` обработки [заданий в очереди](/docs/{{version}}/queues). На практике именно так контейнер должен извлекать большинство ваших объектов.

Например, вы можете объявить сервис, определенный вашим приложением, в конструкторе контроллера. Сервис будет автоматически получен и внедрен в класс:

    <?php

    namespace App\Http\Controllers;

    use App\Services\AppleMusic;

    class PodcastController extends Controller
    {
        /**
         * Создать новый экземпляр контроллера.
         */
        public function __construct(
            protected AppleMusic $apple,
        ) {}

        /**
         * Показать информацию о данном подкасте.
         */
        public function show(string $id): Podcast
        {
            return $this->apple->findPodcast($id);
        }
    }


<a name="method-invocation-and-injection"></a>
## Вызов и внедрение метода

Иногда вам может потребоваться вызвать метод для экземпляра объекта, позволяя контейнеру автоматически вводить зависимости этого метода. Например, учитывая следующий класс:

    <?php

    namespace App;

    use App\Services\AppleMusic;

    class PodcastStats
    {
        /**
         * Generate a new podcast stats report.
         */
        public function generate(AppleMusic $apple): array
        {
            return [
                // ...
            ];
        }
    }

Вы можете вызвать метод `generate` через контейнер следующим образом:

    use App\PodcastStats;
    use Illuminate\Support\Facades\App;

    $stats = App::call([new PodcastStats, 'generate']);

Метод `call` принимает любой вызываемый PHP-код. Метод контейнера `call` может даже использоваться для вызова замыкания при автоматическом внедрении его зависимостей:

    use App\Services\AppleMusic;
    use Illuminate\Support\Facades\App;

    $result = App::call(function (AppleMusic $apple) {
        // ...
    });

<a name="container-events"></a>
## События контейнера

Контейнер служб инициирует событие каждый раз, когда извлекает объект. Вы можете прослушать это событие с помощью метода `resolving`:

    use App\Services\Transistor;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->resolving(Transistor::class, function (Transistor $transistor, Application $app) {
        // Вызывается, когда контейнер извлекает объекты типа `Transistor` ...
    });

    $this->app->resolving(function (mixed $object, Application $app) {
        // Вызывается, когда контейнер извлекает объект любого типа ...
    });

Как видите, извлекаемый объект будет передан в замыкание, что позволит вам установить любые дополнительные свойства объекта до того, как он будет передан его получателю.

<a name="rebinding"></a>
### Перепривязка

Метод `rebinding` позволяет вам прослушивать, когда служба повторно привязывается к контейнеру, то есть она снова регистрируется или переопределяется после первоначальной привязки. Это может быть полезно, когда вам нужно обновить зависимости или изменить поведение каждый раз при обновлении определенной привязки:

    use App\Contracts\PodcastPublisher;
    use App\Services\SpotifyPublisher;
    use App\Services\TransistorPublisher;
    use Illuminate\Contracts\Foundation\Application;

    $this->app->bind(PodcastPublisher::class, SpotifyPublisher::class);

    $this->app->rebinding(
        PodcastPublisher::class,
        function (Application $app, PodcastPublisher $newInstance) {
            //
        },
    );

    // New binding will trigger rebinding closure...
    $this->app->bind(PodcastPublisher::class, TransistorPublisher::class);

<a name="psr-11"></a>
## PSR-11

Контейнер служб Laravel реализует интерфейс [PSR-11](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-11-container). Поэтому вы можете объявить тип интерфейса контейнера PSR-11, чтобы получить экземпляр контейнера Laravel:

    use App\Services\Transistor;
    use Psr\Container\ContainerInterface;

    Route::get('/', function (ContainerInterface $container) {
        $service = $container->get(Transistor::class);

        // ...
    });

Исключение выбрасывается, если данный идентификатор не может быть получен. Исключением будет экземпляр `Psr\Container\NotFoundExceptionInterface`, если идентификатор никогда не был привязан. Если идентификатор был привязан, но не может быть извлечен, будет брошен экземпляр `Psr \ Container \ ContainerExceptionInterface`.
