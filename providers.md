---
git: 910898f77947c560ed9e1017b15314229a4bd1b6
---

# Сервис-провайдеры

<a name="introduction"></a>
## Введение

Сервис-провайдеры – это центральное место начальной загрузки всех приложений Laravel. Ваше собственное приложение, а также все основные службы и сервисы Laravel загружаются через них.

Но, что мы подразумеваем под «начальной загрузкой»? В общем, мы имеем в виду **регистрацию** элементов, включая регистрацию связываний контейнера служб (service container), слушателей событий (event listener), посредников (middleware) и даже маршрутов (route). Сервис-провайдеры являются центральным местом для конфигурирования приложения.

Laravel использует десятки поставщиков услуг внутренне для инициализации своих основных сервисов, таких как почтовый сервис, очереди, кэш и другие. Многие из этих поставщиков являются "отложенными", что означает, что они не будут загружены при каждом запросе, а только когда нужны фактические сервисы, которые они предоставляют.

Все определенные пользователем поставщики услуг регистрируются в файле `bootstrap/providers.php`. В этой документации вы узнаете, как писать собственные сервис-провайдеры и регистрировать их в приложении Laravel.

> [!NOTE]
> Если вы хотите узнать больше о том, как Laravel обрабатывает запросы и работает изнутри, ознакомьтесь с нашей документацией по [жизненному циклу запроса](/docs/{{version}}/lifecycle) Laravel.

<a name="writing-service-providers"></a>
## Написание сервис-провайдеров

Все сервис-провайдеры расширяют класс `Illuminate\Support\ServiceProvider`. Большинство сервис-провайдеров содержат метод `register` и `boot`. В рамках метода `register` следует **только связывать (bind) сущности в [контейнере служб](/docs/{{version}}/container)**. Никогда не следует пытаться зарегистрировать каких-либо слушателей событий, маршруты или что-то другое в методе `register`.

Чтобы сгенерировать новый сервис-провайдер, используйте команду `make:provider` [Artisan](artisan). Laravel автоматически зарегистрирует вашего нового сервис-провайдера в файле `bootstrap/providers.php` вашего приложения:

```shell
php artisan make:provider RiakServiceProvider
```

<a name="the-register-method"></a>
### Метод `register`

Как упоминалось ранее, в рамках метода `register` следует только связывать сущности в [контейнере служб](/docs/{{version}}/container). Никогда не следует пытаться зарегистрировать слушателей событий, маршруты или что-то другое в методе `register`. В противном случае вы можете случайно воспользоваться подсистемой, чей сервис-провайдер еще не загружен.

Давайте взглянем на рядовой сервис-провайдер приложения. В любом из методов сервис-провайдера у вас всегда есть доступ к свойству `$app`, которое обеспечивает доступ к контейнеру служб:

    <?php

    namespace App\Providers;

    use App\Services\Riak\Connection;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Support\ServiceProvider;

    class RiakServiceProvider extends ServiceProvider
    {
        /**
         * Регистрация любых служб приложения.
         */
        public function register(): void
        {
            $this->app->singleton(Connection::class, function (Application $app) {
                return new Connection(config('riak'));
            });
        }
    }

Этот сервис-провайдер определяет только метод `register` и использует этот метод для указания, какая именно реализация `App\Services\Riak\Connection` будет применена в нашем приложении - при помощи контейнера служб. Если вы еще не знакомы с контейнером служб Laravel, ознакомьтесь с [его документацией](/docs/{{version}}/container).

<a name="the-bindings-and-singletons-properties"></a>
#### Свойства `bindings` и `singletons`

Если ваш сервис-провайдер регистрирует много простых связываний, вы можете использовать свойства `bindings` и `singletons` вместо ручной регистрации каждого связывания контейнера. Когда сервис-провайдер загружается фреймворком, он автоматически проверяет эти свойства и регистрирует их связывания:

    <?php

    namespace App\Providers;

    use App\Contracts\DowntimeNotifier;
    use App\Contracts\ServerProvider;
    use App\Services\DigitalOceanServerProvider;
    use App\Services\PingdomDowntimeNotifier;
    use App\Services\ServerToolsProvider;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Все связывания контейнера, которые должны быть зарегистрированы.
         *
         * @var array
         */
        public $bindings = [
            ServerProvider::class => DigitalOceanServerProvider::class,
        ];

        /**
         * Все синглтоны контейнера, которые должны быть зарегистрированы.
         *
         * @var array
         */
        public $singletons = [
            DowntimeNotifier::class => PingdomDowntimeNotifier::class,
            ServerProvider::class => ServerToolsProvider::class,
        ];
    }

<a name="the-boot-method"></a>
### Метод `boot`

Итак, что, если нам нужно зарегистрировать [компоновщик шаблонов](/docs/{{version}}/views#view-composers) в нашем сервис-провайдере? Это должно быть сделано в рамках метода `boot`. **Этот метод вызывается после регистрации всех остальных сервис-провайдеров**, что означает, что в этом месте у вас уже есть доступ ко всем другим службам, которые были зарегистрированы фреймворком:

    <?php

    namespace App\Providers;

    use Illuminate\Support\Facades\View;
    use Illuminate\Support\ServiceProvider;

    class ComposerServiceProvider extends ServiceProvider
    {
        /**
         * Загрузка любых служб приложения.
         */
        public function boot(): void
        {
            View::composer('view', function () {
                // ...
            });
        }
    }

<a name="boot-method-dependency-injection"></a>
#### Внедрение зависимости в методе `boot`

Вы можете указывать тип зависимостей в методе `boot` сервис-провайдера. [Контейнер служб](/docs/{{version}}/container) автоматически внедрит любые необходимые зависимости:

    use Illuminate\Contracts\Routing\ResponseFactory;

    /**
     * Загрузка любых служб приложения.
     */
    public function boot(ResponseFactory $response): void
    {
        $response->macro('serialized', function (mixed $value) {
            // ...
        });
    }

<a name="registering-providers"></a>
## Регистрация сервис-провайдеров

Все поставщики услуг регистрируются в файле конфигурации `bootstrap/providers.php`. Этот файл возвращает массив, который содержит имена классов поставщиков услуг вашего приложения:

    <?php

    return [
        App\Providers\AppServiceProvider::class,
    ];

Когда вы вызываете команду `make:provider` в Artisan, Laravel автоматически добавит сгенерированный провайдер в файл `bootstrap/providers.php`. Однако, если вы создали класс провайдера вручную, вы должны вручную добавить класс провайдера в массив:

    <?php

    return [
        App\Providers\AppServiceProvider::class,
        App\Providers\ComposerServiceProvider::class, // [tl! add]
    ];

<a name="deferred-providers"></a>
## Отложенные сервис-провайдеры

Если ваш сервис-провайдер регистрирует **только** связывания в [контейнере служб](/docs/{{version}}/container), вы можете отложить его регистрацию до тех пор, пока одно из зарегистрированных связываний не понадобится. Отсрочка загрузки такого сервис-провайдера повысит производительность вашего приложения, так как он не загружается из файловой системы при каждом запросе.

Laravel составляет и сохраняет список всех служб, предоставляемых отложенными сервис-провайдерами, а также имя класса сервис-провайдера. Laravel загрузит сервис-провайдер только при необходимости в одной из этих служб.

Чтобы отложить загрузку сервис-провайдера, реализуйте интерфейс `\Illuminate\Contracts\Support\DeferrableProvider`, описав метод `provides`. Метод `provides` должен вернуть связывания контейнера службы, регистрируемые данным классом:

    <?php

    namespace App\Providers;

    use App\Services\Riak\Connection;
    use Illuminate\Contracts\Foundation\Application;
    use Illuminate\Contracts\Support\DeferrableProvider;
    use Illuminate\Support\ServiceProvider;

    class RiakServiceProvider extends ServiceProvider implements DeferrableProvider
    {
        /**
         * Регистрация любых служб приложения.
         */
        public function register(): void
        {
            $this->app->singleton(Connection::class, function (Application $app) {
                return new Connection($app['config']['riak']);
            });
        }

        /**
         * Получить службы, предоставляемые поставщиком.
         *
         * @return array<int, string>
         */
        public function provides(): array
        {
            return [Connection::class];
        }
    }
