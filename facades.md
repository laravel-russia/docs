---
git: 1a3d112255c8d0da2cbc10ebe581deebdd172ae2
---

# Фасады (Facades)

<a name="introduction"></a>
## Введение

В документации Laravel вы увидите примеры кода, демонстрирующего взаимодействия с функционалом Laravel через «фасады». Фасады предоставляют «статический» интерфейс для классов, доступных в [контейнере служб](/docs/{{version}}/container) приложения. Laravel из коробки включает множество фасадов, обеспечивающих доступ почти ко всему функционалу Laravel.

Фасады Laravel служат «статическими прокси» для базовых классов в контейнере служб, обеспечивая преимущества краткого, выразительного синтаксиса при сохранении большей тестируемости и гибкости, чем традиционные статические методы. Если вы не совсем понимаете, как фасады работают под капотом просто идите вперед – оставьте этот момент на будущее, примите к сведению синтаксис фасадов и то, что они похожи на статические вызовы, и просто продолжайте изучать Laravel.

Все фасады Laravel определены в пространстве имён `Illuminate\Support\Facades`. Таким образом, мы можем легко получить доступ к такому фасаду:

    use Illuminate\Support\Facades\Cache;
    use Illuminate\Support\Facades\Route;

    Route::get('/cache', function () {
        return Cache::get('key');
    });

В документации Laravel во многих примерах будут использоваться фасады для демонстрации различного функционала фреймворка.

<a name="helper-functions"></a>
#### Глобальные помощники

В дополнении к фасадам, Laravel предлагает множество глобальных «вспомогательных функций», которые упрощают взаимодействие с общими функциями Laravel. Вот некоторые из глобальных помощников, с которыми вы можете взаимодействовать – это `view`, `response`, `url`, `config` и т.д. Каждый помощник, предлагаемый Laravel, задокументирован с соответствующей функцией; однако полный список доступен в специальной [документации глобальных помощников](helpers).

Например, вместо использования фасада `Illuminate\Support\Facades\Response` для генерации ответа JSON, мы можем просто использовать функцию `response`. Поскольку помощники доступны глобально, то вам не нужно импортировать какие-либо классы, чтобы использовать их:

    use Illuminate\Support\Facades\Response;

    Route::get('/users', function () {
        return Response::json([
            // ...
        ]);
    });

    Route::get('/users', function () {
        return response()->json([
            // ...
        ]);
    });

<a name="when-to-use-facades"></a>
## Когда использовать фасады

У фасадов много преимуществ. Они предоставляют краткий, запоминающийся синтаксис, позволяющий вам использовать функции Laravel, не запоминая длинные имена классов, которые необходимо вводить или конфигурировать вручную. Более того, благодаря уникальному использованию динамических методов PHP их легко протестировать.

Однако при использовании фасадов необходимо соблюдать некоторую осторожность. Основная опасность фасадов – «разрастание» класса. Поскольку фасады настолько просты в использовании и не требуют внедрений, что легко сказывается на разрастании класса и использовании множества фасадов в одном классе. При использовании внедрения зависимостей этот потенциал снижается за счет визуальной обратной связи, которую дает большой конструктор, сигнализируя о том, что ваш класс становится слишком большим. Поэтому, используя фасады, обратите особое внимание на размер вашего класса, чтобы уровень его ответственности оставался узким. Если ваш класс становится слишком большим, рассмотрите возможность разделения его на несколько более мелких классов.

<a name="facades-vs-dependency-injection"></a>
### Фасады против внедрения зависимостей

Одним из основных преимуществ внедрения зависимостей является возможность изменения реализации внедренного класса. Это полезно во время тестирования, так как вы можете вставить имитацию или заглушку и утверждать, что для заглушки были вызваны различные методы.

Как правило, невозможно имитировать или заглушить действительно статический метод класса. Однако, поскольку фасады используют динамические методы для проксирования вызовов методов к объектам, извлекаемым из контейнера служб, мы фактически можем тестировать фасады так же, как тестировали бы внедренный экземпляр класса. Например, учитывая следующий маршрут:

    use Illuminate\Support\Facades\Cache;

    Route::get('/cache', function () {
        return Cache::get('key');
    });

Используя методы тестирования фасадов Laravel, мы можем написать следующий тест, чтобы проверить, что метод `Cache::get` был вызван с ожидаемым аргументом:

```php tab=Pest
use Illuminate\Support\Facades\Cache;

test('базовый пример', function () {
    Cache::shouldReceive('get')
         ->with('key')
         ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
});
```

```php tab=PHPUnit
use Illuminate\Support\Facades\Cache;

/**
 * Пример базового функционального теста.
 */
public function test_basic_example(): void
{
    Cache::shouldReceive('get')
         ->with('key')
         ->andReturn('value');

    $response = $this->get('/cache');

    $response->assertSee('value');
}
```

<a name="facades-vs-helper-functions"></a>
### Фасады против глобальных помощников

Помимо фасадов, Laravel включает в себя множество «вспомогательных» функций, которые могут выполнять общие задачи, такие как генерация шаблонов, запуск событий, запуск заданий или отправка HTTP-ответов. Многие из этих вспомогательных функций выполняют ту же функцию, что и соответствующий фасад. Например, этот вызов фасада и вызов помощника эквивалентны:

    return Illuminate\Support\Facades\View::make('profile');

    return view('profile');

Практической разницы между фасадами и глобальными помощниками нет абсолютно никакой. При использовании глобальных помощников вы все равно можете тестировать их точно так же, как и соответствующий фасад. Например, учитывая следующий маршрут:

    Route::get('/cache', function () {
        return cache('key');
    });

Помощник `cache` будет вызывать метод `get` в базовом классе, лежащем в основе фасада `Cache`. Таким образом, даже если мы используем вспомогательную функцию, мы можем написать следующий тест, чтобы убедиться, что метод был вызван с ожидаемым аргументом:

    use Illuminate\Support\Facades\Cache;

    /**
     * Отвлеченный пример функционального теста.
     */
    public function test_basic_example(): void
    {
        Cache::shouldReceive('get')
             ->with('key')
             ->andReturn('value');

        $response = $this->get('/cache');

        $response->assertSee('value');
    }

<a name="how-facades-work"></a>
## Как фасады работают

В приложении Laravel фасад – это класс, который обеспечивает доступ к объекту из контейнера. Техника, которая выполняет эту работу, относится к классу `Facade`. Фасады Laravel и любые пользовательские фасады, которые вы создаете, будут расширять базовый класс `Illuminate\Support\Facades\Facade`.

Базовый класс `Facade` использует магический метод `__callStatic()`, чтобы делегировать вызовы с вашего фасада объекту, извлеченному из контейнера. В приведенном ниже примере выполняется вызов кеш-системы Laravel. Взглянув на этот код, можно предположить, что статический метод `get` вызывается в классе `Cache`:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\Cache;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * Показать профиль конкретного пользователя.
         */
        public function showProfile(string $id): View
        {
            $user = Cache::get('user:'.$id);

            return view('profile', ['user' => $user]);
        }
    }

Обратите внимание, что в верхней части файла мы «импортируем» фасад `Cache`. Этот фасад служит прокси для доступа к базовой реализации интерфейса `Illuminate\Contracts\Cache\Factory`. Любые вызовы, которые мы делаем с использованием фасада, будут переданы в базовый экземпляр службы кеширования Laravel.

Если мы посмотрим на этот класс `Illuminate\Support\Facades\Cache`, вы увидите, что статического метода `get` не существует:

    class Cache extends Facade
    {
        /**
         * Получить зарегистрированное имя компонента.
         */
        protected static function getFacadeAccessor(): string
        {
            return 'cache';
        }
    }

Вместо этого фасад `Cache` расширяет базовый класс `Facade` и определяет метод `getFacadeAccessor()`. Задача этого метода – вернуть имя привязки контейнера службы. Когда пользователь ссылается на любой статический метод фасада `Cache`, Laravel извлекает объект из [контейнера служб](/docs/{{version}}/container), привязанный к `cache` и запускает запрошенный метод (в данном случае `get`) этого объекта.

<a name="real-time-facades"></a>
## Фасады в реальном времени

Используя фасады в реальном времени, вы можете рассматривать любой класс в своем приложении, как если бы он был фасадом. Чтобы проиллюстрировать, как это можно использовать, давайте сначала рассмотрим код, который не использует фасады в реальном времени. Например, предположим, что наша модель `Podcast` имеет метод `publish`. Однако, чтобы опубликовать подкаст, нам нужно внедрить экземпляр `Publisher`:

    <?php

    namespace App\Models;

    use App\Contracts\Publisher;
    use Illuminate\Database\Eloquent\Model;

    class Podcast extends Model
    {
        /**
         * Опубликовать подкаст.
         */
        public function publish(Publisher $publisher): void
        {
            $this->update(['publishing' => now()]);

            $publisher->publish($this);
        }
    }

Внедрение реализации издателя (`Publisher`) в метод позволяет нам легко тестировать метод изолированно, поскольку мы можем имитировать внедренного издателя. Однако он требует от нас всегда передавать экземпляр издателя каждый раз, когда мы вызываем метод `publish`. Используя фасады в реальном времени, мы можем поддерживать такую же тестируемость, при этом не требуя явной передачи экземпляра `Publisher`. Чтобы сгенерировать фасад в реальном времени, добавьте к пространству имен импортируемого класса префикс `Facades`:

    <?php

    namespace App\Models;

    use App\Contracts\Publisher; // [tl! remove]
    use Facades\App\Contracts\Publisher; // [tl! add]
    use Illuminate\Database\Eloquent\Model;

    class Podcast extends Model
    {
        /**
         * Опубликовать подкаст.
         */
        public function publish(Publisher $publisher): void // [tl! remove]
        public function publish(): void // [tl! add]
        {
            $this->update(['publishing' => now()]);

            $publisher->publish($this); // [tl! remove]
            Publisher::publish($this); // [tl! add]
        }
    }

Когда используется фасад реального времени, реализация издателя будет получена из контейнера службы с использованием той части интерфейса или имени класса, которая расположена после префикса `Facades`. При тестировании мы можем использовать встроенные в Laravel помощники для тестирования фасадов, чтобы имитировать вызов этого метода:

```php tab=Pest
<?php

use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('подкаст может быть опубликован', function () {
    $podcast = Podcast::factory()->create();

    Publisher::shouldReceive('publish')->once()->with($podcast);

    $podcast->publish();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Models\Podcast;
use Facades\App\Contracts\Publisher;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PodcastTest extends TestCase
{
    use RefreshDatabase;

    /**
     * Тестовый пример.
     */
    public function test_podcast_can_be_published(): void
    {
        $podcast = Podcast::factory()->create();

        Publisher::shouldReceive('publish')->once()->with($podcast);

        $podcast->publish();
    }
}
```

<a name="facade-class-reference"></a>
## Справочник фасадов

Ниже вы найдете каждый фасад и его базовый класс. Это полезный инструмент для быстрого поиска в документации API. Ключ [привязки в контейнере служб](/docs/{{version}}/container) также указан, где это возможно.

| Фасад                 | Класс                                                                                                                                          | Привязка в контейнере служб |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------- |
| App                   | [Illuminate\\Foundation\\Application](https://laravel.com/api/{{version}}/Illuminate/Foundation/Application.html)                              | `app`                       |
| Artisan               | [Illuminate\\Contracts\\Console\\Kernel](https://laravel.com/api/{{version}}/Illuminate/Contracts/Console/Kernel.html)                         | `artisan`                   |
| Auth (Instance)       | [Illuminate\\Contracts\\Auth\\Guard](https://laravel.com/api/{{version}}/Illuminate/Contracts/Auth/Guard.html)                                 | `auth.driver`               |
| Auth                  | [Illuminate\\Auth\\AuthManager](https://laravel.com/api/{{version}}/Illuminate/Auth/AuthManager.html)                                          | `auth`                      |
| Blade                 | [Illuminate\\View\\Compilers\\BladeCompiler](https://laravel.com/api/{{version}}/Illuminate/View/Compilers/BladeCompiler.html)                 | `blade.compiler`            |
| Broadcast (Instance)  | [Illuminate\\Contracts\\Broadcasting\\Broadcaster](https://laravel.com/api/{{version}}/Illuminate/Contracts/Broadcasting/Broadcaster.html)     | &nbsp;                      |
| Broadcast             | [Illuminate\\Contracts\\Broadcasting\\Factory](https://laravel.com/api/{{version}}/Illuminate/Contracts/Broadcasting/Factory.html)             | &nbsp;                      |
| Bus                   | [Illuminate\\Contracts\\Bus\\Dispatcher](https://laravel.com/api/{{version}}/Illuminate/Contracts/Bus/Dispatcher.html)                         | &nbsp;                      |
| Cache (Instance)      | [Illuminate\\Cache\\Repository](https://laravel.com/api/{{version}}/Illuminate/Cache/Repository.html)                                          | `cache.store`               |
| Cache                 | [Illuminate\\Cache\\CacheManager](https://laravel.com/api/{{version}}/Illuminate/Cache/CacheManager.html)                                      | `cache`                     |
| Config                | [Illuminate\\Config\\Repository](https://laravel.com/api/{{version}}/Illuminate/Config/Repository.html)                                        | `config`                    |
| Context               | [Illuminate\\Log\\Context\\Repository](https://laravel.com/api/{{version}}/Illuminate/Log/Context/Repository.html)                             | &nbsp;                      |
| Cookie                | [Illuminate\\Cookie\\CookieJar](https://laravel.com/api/{{version}}/Illuminate/Cookie/CookieJar.html)                                          | `cookie`                    |
| Crypt                 | [Illuminate\\Encryption\\Encrypter](https://laravel.com/api/{{version}}/Illuminate/Encryption/Encrypter.html)                                  | `encrypter`                 |
| Date                  | [Illuminate\\Support\\DateFactory](https://laravel.com/api/{{version}}/Illuminate/Support/DateFactory.html)                                    | `date`                      |
| DB (Instance)         | [Illuminate\\Database\\Connection](https://laravel.com/api/{{version}}/Illuminate/Database/Connection.html)                                    | `db.connection`             |
| DB                    | [Illuminate\\Database\\DatabaseManager](https://laravel.com/api/{{version}}/Illuminate/Database/DatabaseManager.html)                          | `db`                        |
| Event                 | [Illuminate\\Events\\Dispatcher](https://laravel.com/api/{{version}}/Illuminate/Events/Dispatcher.html)                                        | `events`                    |
| Exceptions (Instance) | [Illuminate\\Contracts\\Debug\\ExceptionHandler](https://laravel.com/api/{{version}}/Illuminate/Contracts/Debug/ExceptionHandler.html)         | &nbsp;                      |
| Exceptions            | [Illuminate\\Foundation\\Exceptions\\Handler](https://laravel.com/api/{{version}}/Illuminate/Foundation/Exceptions/Handler.html)               | &nbsp;                      |
| File                  | [Illuminate\\Filesystem\\Filesystem](https://laravel.com/api/{{version}}/Illuminate/Filesystem/Filesystem.html)                                | `files`                     |
| Gate                  | [Illuminate\\Contracts\\Auth\\Access\\Gate](https://laravel.com/api/{{version}}/Illuminate/Contracts/Auth/Access/Gate.html)                    | &nbsp;                      |
| Hash                  | [Illuminate\\Contracts\\Hashing\\Hasher](https://laravel.com/api/{{version}}/Illuminate/Contracts/Hashing/Hasher.html)                         | `hash`                      |
| Http                  | [Illuminate\\Http\\Client\\Factory](https://laravel.com/api/{{version}}/Illuminate/Http/Client/Factory.html)                                   | &nbsp;                      |
| Lang                  | [Illuminate\\Translation\\Translator](https://laravel.com/api/{{version}}/Illuminate/Translation/Translator.html)                              | `translator`                |
| Log                   | [Illuminate\\Log\\LogManager](https://laravel.com/api/{{version}}/Illuminate/Log/LogManager.html)                                              | `log`                       |
| Mail                  | [Illuminate\\Mail\\Mailer](https://laravel.com/api/{{version}}/Illuminate/Mail/Mailer.html)                                                    | `mailer`                    |
| Notification          | [Illuminate\\Notifications\\ChannelManager](https://laravel.com/api/{{version}}/Illuminate/Notifications/ChannelManager.html)                  | &nbsp;                      |
| Password (Instance)   | [Illuminate\\Auth\\Passwords\\PasswordBroker](https://laravel.com/api/{{version}}/Illuminate/Auth/Passwords/PasswordBroker.html)               | `auth.password.broker`      |
| Password              | [Illuminate\\Auth\\Passwords\\PasswordBrokerManager](https://laravel.com/api/{{version}}/Illuminate/Auth/Passwords/PasswordBrokerManager.html) | `auth.password`             |
| Pipeline (Instance)   | [Illuminate\\Pipeline\\Pipeline](https://laravel.com/api/{{version}}/Illuminate/Pipeline/Pipeline.html)                                        | &nbsp;                      |
| Process               | [Illuminate\\Process\\Factory](https://laravel.com/api/{{version}}/Illuminate/Process/Factory.html)                                            | &nbsp;                      |
| Queue (Base Class)    | [Illuminate\\Queue\\Queue](https://laravel.com/api/{{version}}/Illuminate/Queue/Queue.html)                                                    | &nbsp;                      |
| Queue (Instance)      | [Illuminate\\Contracts\\Queue\\Queue](https://laravel.com/api/{{version}}/Illuminate/Contracts/Queue/Queue.html)                               | `queue.connection`          |
| Queue                 | [Illuminate\\Queue\\QueueManager](https://laravel.com/api/{{version}}/Illuminate/Queue/QueueManager.html)                                      | `queue`                     |
| RateLimiter           | [Illuminate\\Cache\\RateLimiter](https://laravel.com/api/{{version}}/Illuminate/Cache/RateLimiter.html)                                        | &nbsp;                      |
| Redirect              | [Illuminate\\Routing\\Redirector](https://laravel.com/api/{{version}}/Illuminate/Routing/Redirector.html)                                      | `redirect`                  |
| Redis (Instance)      | [Illuminate\\Redis\\Connections\\Connection](https://laravel.com/api/{{version}}/Illuminate/Redis/Connections/Connection.html)                 | `redis.connection`          |
| Redis                 | [Illuminate\\Redis\\RedisManager](https://laravel.com/api/{{version}}/Illuminate/Redis/RedisManager.html)                                      | `redis`                     |
| Request               | [Illuminate\\Http\\Request](https://laravel.com/api/{{version}}/Illuminate/Http/Request.html)                                                  | `request`                   |
| Response (Instance)   | [Illuminate\\Http\\Response](https://laravel.com/api/{{version}}/Illuminate/Http/Response.html)                                                | &nbsp;                      |
| Response              | [Illuminate\\Contracts\\Routing\\ResponseFactory](https://laravel.com/api/{{version}}/Illuminate/Contracts/Routing/ResponseFactory.html)       | &nbsp;                      |
| Route                 | [Illuminate\\Routing\\Router](https://laravel.com/api/{{version}}/Illuminate/Routing/Router.html)                                              | `router`                    |
| Schedule              | [Illuminate\\Console\\Scheduling\\Schedule](https://laravel.com/api/{{version}}/Illuminate/Console/Scheduling/Schedule.html)                   | &nbsp;                      |
| Schema                | [Illuminate\\Database\\Schema\\Builder](https://laravel.com/api/{{version}}/Illuminate/Database/Schema/Builder.html)                           | &nbsp;                      |
| Session (Instance)    | [Illuminate\\Session\\Store](https://laravel.com/api/{{version}}/Illuminate/Session/Store.html)                                                | `session.store`             |
| Session               | [Illuminate\\Session\\SessionManager](https://laravel.com/api/{{version}}/Illuminate/Session/SessionManager.html)                              | `session`                   |
| Storage (Instance)    | [Illuminate\\Contracts\\Filesystem\\Filesystem](https://laravel.com/api/{{version}}/Illuminate/Contracts/Filesystem/Filesystem.html)           | `filesystem.disk`           |
| Storage               | [Illuminate\\Filesystem\\FilesystemManager](https://laravel.com/api/{{version}}/Illuminate/Filesystem/FilesystemManager.html)                  | `filesystem`                |
| URL                   | [Illuminate\\Routing\\UrlGenerator](https://laravel.com/api/{{version}}/Illuminate/Routing/UrlGenerator.html)                                  | `url`                       |
| Validator (Instance)  | [Illuminate\\Validation\\Validator](https://laravel.com/api/{{version}}/Illuminate/Validation/Validator.html)                                  | &nbsp;                      |
| Validator             | [Illuminate\\Validation\\Factory](https://laravel.com/api/{{version}}/Illuminate/Validation/Factory.html)                                      | `validator`                 |
| View (Instance)       | [Illuminate\\View\\View](https://laravel.com/api/{{version}}/Illuminate/View/View.html)                                                        | &nbsp;                      |
| View                  | [Illuminate\\View\\Factory](https://laravel.com/api/{{version}}/Illuminate/View/Factory.html)                                                  | `view`                      |
| Vite                  | [Illuminate\\Foundation\\Vite](https://laravel.com/api/{{version}}/Illuminate/Foundation/Vite.html)                                            | &nbsp;                      |
