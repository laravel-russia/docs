---
git: 933bda5f2aa87567d24096f8967a4c287ee6b9f3
---


# Laravel Pennant

<a name="introduction"></a>
## Введение

[Laravel Pennant](https://github.com/laravel/pennant) - это простой и легковесный пакет для управления флагами функций, без лишних наворотов. Флаги функций позволяют вам с уверенностью постепенно внедрять новые возможности приложения, проводить A/B-тестирование новых дизайнов интерфейса, дополнять стратегию разработки на основе ветвей и многое другое.

<a name="installation"></a>
## Установка

Сначала установите Pennant в свой проект, используя менеджер пакетов Composer:

```shell
composer require laravel/pennant
```

Затем опубликуйте файлы конфигурации и миграции Pennant с помощью команды Artisan `vendor:publish`:

```shell
php artisan vendor:publish --provider="Laravel\Pennant\PennantServiceProvider"
```

Наконец, запустите миграции базы данных вашего приложения. Это создаст таблицу `features`, которую Pennant использует для работы с драйвером `database`:

```shell
php artisan migrate
```

<a name="configuration"></a>
## Настройка

После публикации ресурсов Pennant, его файл конфигурации будет находиться по пути `config/pennant.php`. В этом файле конфигурации вы можете указать механизм хранения по умолчанию, который будет использоваться Pennant для сохранения определённых значений флагов функций.

Pennant включает поддержку хранения определённых значений флагов функций в памяти массива с помощью драйвера `array`. Также Pennant может сохранять определенные значения флагов функций постоянно в реляционной базе данных с помощью драйвера `database`, который является механизмом хранения по умолчанию, используемым Pennant.

<a name="defining-features"></a>
## Определение функций

Для определения функции вы можете использовать метод `define`, предоставляемый фасадом `Feature`. Вам потребуется указать имя функции, а также замыкание, которое будет вызвано для определения начального значения функции.

Обычно функции определяются в сервис-провайдере с использованием фасада `Feature`. Замыкание будет получать "область" (scope) для проверки функции. Наиболее часто используемой областью является текущий аутентифицированный пользователь. В этом примере мы определим функцию для поэтапного внедрения нового API для пользователей нашего приложения:

```php
<?php

namespace App\Providers;

use App\Models\User;
use Illuminate\Support\Lottery;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Feature::define('new-api', fn (User $user) => match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        });
    }
}
```

Как видите, у нас есть следующие правила для нашей функции:

- Все внутренние члены команды должны использовать новое API.
- Любые клиенты с высоким трафиком не должны использовать новое API.
- В остальных случаях функция должна быть активна для пользователей с вероятностью 1 к 100.

При первой проверке функции `new-api` для определенного пользователя результат замыкания будет сохранен драйвером хранилища. При следующей проверке функции для того же пользователя значение будет извлечено из хранилища, и замыкание не будет вызвано.

Для удобства, если определение функции возвращает только лотерею, вы можете полностью опустить замыкание:

    Feature::define('site-redesign', Lottery::odds(1, 1000));

<a name="class-based-features"></a>
### Определение функций на основе классов

Pennant также позволяет вам определять функции на основе классов. В отличие от определений функций на основе замыканий, нет необходимости регистрировать функцию на основе класса в сервис-провайдере. Для создания функции на основе класса вы можете использовать команду Artisan `pennant:feature`. По умолчанию класс функции будет размещен в директории `app/Features` вашего приложения:

```shell
php artisan pennant:feature NewApi
```

При написании класса функции вам нужно определить только метод `resolve`, который будет вызван для определения начального значения функции для данной области. Опять же, обычно областью будет текущий аутентифицированный пользователь:

```php
<?php

namespace App\Features;

use App\Models\User;
use Illuminate\Support\Lottery;

class NewApi
{
    /**
     * Определите начальное значение функции.
     */
    public function resolve(User $user): mixed
    {
        return match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        };
    }
}
```

Если вы хотите вручную разрешить экземпляр функции на основе класса, вы можете вызвать метод `instance` на фасаде `Feature`:

```php
use Illuminate\Support\Facades\Feature;

$instance = Feature::instance(NewApi::class);
```

> [!NOTE]
> Классы функций разрешаются через [контейнер](/docs/{{version}}/container), поэтому при необходимости вы можете внедрять зависимости в конструктор класса функции.

#### Настройка хранимого имени функции

По умолчанию Pennant будет хранить полное имя класса функции. Если вы хотите отделить хранимое имя функции от внутренней структуры приложения, вы можете указать свойство `$name` в классе функции. Значение этого свойства будет храниться вместо имени класса:

```php
<?php

namespace App\Features;

class NewApi
{
    /**
    * Сохраненное имя функции.
     *
     * @var string
     */
    public $name = 'new-api';

    // ...
}
```

<a name="checking-features"></a>
## Проверка функций

Чтобы определить, активна ли функция, вы можете использовать метод `active` фасада `Feature`. По умолчанию функции проверяются относительно текущего аутентифицированного пользователя:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * Отобразить список ресурсов.
     */
    public function index(Request $request): Response
    {
        return Feature::active('new-api')
                ? $this->resolveNewApiResponse($request)
                : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

Хотя  по умолчанию функции проверяются относительно текущего аутентифицированного пользователя, вы можете легко проверить функцию относительно другого пользователя или [области](#scope). Для этого используйте метод `for`, предоставляемый фасадом `Feature`:

```php
return Feature::for($user)->active('new-api')
        ? $this->resolveNewApiResponse($request)
        : $this->resolveLegacyApiResponse($request);
```

Pennant также предлагает несколько дополнительных удобных методов, которые могут оказаться полезными при определении активна ли функция или нет:

```php
// Определить, активны ли все указанные функции...
Feature::allAreActive(['new-api', 'site-redesign']);

// Определить, активна ли хотя бы одна из указанных функций...
Feature::someAreActive(['new-api', 'site-redesign']);

// Определить, неактивна ли функция...
Feature::inactive('new-api');

// Определить, неактивны ли все указанные функции...
Feature::allAreInactive(['new-api', 'site-redesign']);

// Определить, неактивна ли хотя бы одна из указанных функций...
Feature::someAreInactive(['new-api', 'site-redesign']);
```

> [!NOTE]
> При использовании Pennant вне контекста HTTP, например в команде Artisan или в задании в очереди, обычно следует [явно указывать область функции](#specifying-the-scope). В качестве альтернативы вы можете определить [область по умолчанию](#default-scope), которая учитывает как аутентифицированные контексты HTTP, так и неаутентифицированные контексты.

<a name="checking-class-based-features"></a>
#### Проверка функций на основе классов

Для функций на основе классов вы должны предоставить имя класса при проверке функции:

```php
<?php

namespace App\Http\Controllers;

use App\Features\NewApi;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Feature;

class PodcastController
{
    /**
     * Отобразить список ресурсов.
     */
    public function index(Request $request): Response
    {
        return Feature::active(NewApi::class)
                ? $this->resolveNewApiResponse($request)
                : $this->resolveLegacyApiResponse($request);
    }

    // ...
}
```

<a name="conditional-execution"></a>
### Условное выполнение

Метод `when` может быть использован для плавного выполнения заданного замыкания, если функция активна. Кроме того, можно предоставить второе замыкание, которое будет выполнено, если функция неактивна:

    <?php

    namespace App\Http\Controllers;

    use App\Features\NewApi;
    use Illuminate\Http\Request;
    use Illuminate\Http\Response;
    use Laravel\Pennant\Feature;

    class PodcastController
    {
        /**
         * Отобразить список ресурсов.
         */
        public function index(Request $request): Response
        {
            return Feature::when(NewApi::class,
                fn () => $this->resolveNewApiResponse($request),
                fn () => $this->resolveLegacyApiResponse($request),
            );
        }

        // ...
    }

Метод `unless` является противоположностью метода `when`, он выполняет первое замыкание, если функция неактивна:

    return Feature::unless(NewApi::class,
        fn () => $this->resolveLegacyApiResponse($request),
        fn () => $this->resolveNewApiResponse($request),
    );

<a name="the-has-features-trait"></a>
### Трейт `HasFeatures`

Трейт `HasFeatures` из Pennant может быть добавлен к модели `User` вашего приложения (или любой другой модели, у которой есть функции), чтобы предоставить удобный способ проверки функций напрямую из модели:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Pennant\Concerns\HasFeatures;

class User extends Authenticatable
{
    use HasFeatures;

    // ...
}
```

После добавления трейта к вашей модели вы можете легко проверить функции, вызвав метод `features`:

```php
if ($user->features()->active('new-api')) {
    // ...
}
```

Разумеется, метод `features` предоставляет доступ ко многим другим удобным методам взаимодействия с функциями:

```php
// Значения...
$value = $user->features()->value('purchase-button')
$values = $user->features()->values(['new-api', 'purchase-button']);

// Состояние...
$user->features()->active('new-api');
$user->features()->allAreActive(['new-api', 'server-api']);
$user->features()->someAreActive(['new-api', 'server-api']);

$user->features()->inactive('new-api');
$user->features()->allAreInactive(['new-api', 'server-api']);
$user->features()->someAreInactive(['new-api', 'server-api']);

// Условное выполнение...
$user->features()->when('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);

$user->features()->unless('new-api',
    fn () => /* ... */,
    fn () => /* ... */,
);
```

<a name="blade-directive"></a>
### Директива Blade

Для более удобной проверки функций в Blade Pennant предлагает директивы `@feature` и `@featureany`:

```blade
@feature('site-redesign')
    <!-- 'site-redesign' активен -->
@else
    <!-- 'site-redesign' неактивен -->
@endfeature

@featureany(['site-redesign', 'beta'])
    <!-- 'site-redesign' или `beta` активен -->
@endfeatureany
```

<a name="middleware"></a>
### Middleware

Pennant также включает [middleware](/docs/{{version}}/middleware), которое можно использовать для проверки доступа текущего аутентифицированного пользователя к функции до вызова маршрута. Вы можете назначить middleware маршруту и указать функции, которые требуются для доступа к маршруту. Если хотя бы одна из указанных функций неактивна для текущего аутентифицированного пользователя, маршрут вернет ответ HTTP `400 Bad Request`. Методу `using` в качестве параметров могут быть переданы несколько функций.

```php
use Illuminate\Support\Facades\Route;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

Route::get('/api/servers', function () {
    // ...
})->middleware(EnsureFeaturesAreActive::using('new-api', 'servers-api'));
```

<a name="customizing-the-response"></a>
#### Настройка ответа

Если вы хотите настроить ответ, который возвращает middleware, когда одна из указанных функций неактивна, вы можете использовать метод `whenInactive`, предоставленный middleware `EnsureFeaturesAreActive`. Обычно этот метод следует вызывать в методе `boot` одного из сервис-провайдеров вашего приложения:

```php
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Laravel\Pennant\Middleware\EnsureFeaturesAreActive;

/**
 * Загрузка любых сервисов приложения.
 */
public function boot(): void
{
    EnsureFeaturesAreActive::whenInactive(
        function (Request $request, array $features) {
            return new Response(status: 403);
        }
    );

    // ...
}
```

<a name="intercepting-feature-checks"></a>
### Перехват проверок свойства

Иногда может быть полезно выполнить некоторые проверки в памяти перед получением сохраненного значения данного свойства. Представьте, что вы разрабатываете новый API на основе флага свойства и хотите иметь возможность отключить новый API, не теряя ни одного из разрешенных значений свойства в хранилище. Если вы заметили ошибку в новом API, вы можете легко отключить его для всех, кроме членов внутренней команды, исправить ошибку, а затем повторно включить новый API для пользователей, которые ранее имели доступ к этому свойству.

Этого можно добиться с помощью метода `before` [class-based Feature's](#class-based-features). Если метод `before` присутствует, он всегда запускается в памяти перед извлечением значения из хранилища. Если из метода возвращается значение, отличное от null, оно будет использоваться вместо сохраненного значения функции на время запроса:

```php
<?php

namespace App\Features;

use App\Models\User;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Lottery;

class NewApi
{
    /**
     * Запустите проверку «всегда в памяти» перед извлечением сохраненного значения.
     */
    public function before(User $user): mixed
    {
        if (Config::get('features.new-api.disabled')) {
            return $user->isInternalTeamMember();
        }
    }

    /**
     * Определите начальное значение функции.
     */
    public function resolve(User $user): mixed
    {
        return match (true) {
            $user->isInternalTeamMember() => true,
            $user->isHighTrafficCustomer() => false,
            default => Lottery::odds(1 / 100),
        };
    }
}
```

Вы также можете использовать эту функцию, чтобы запланировать глобальное развертывание функции, которая ранее находилась за флажком свойства:

```php
<?php

namespace App\Features;

use Illuminate\Support\Carbon;
use Illuminate\Support\Facades\Config;

class NewApi
{
    /**
     * Запустите проверку «всегда в памяти» перед извлечением сохраненного значения.
     */
    public function before(User $user): mixed
    {
        if (Config::get('features.new-api.disabled')) {
            return $user->isInternalTeamMember();
        }

        if (Carbon::parse(Config::get('features.new-api.rollout-date'))->isPast()) {
            return true;
        }
    }

    // ...
}
```

<a name="in-memory-cache"></a>
### Кэш в памяти

При проверке функции Pennant создаст кеш результата в памяти. Если вы используете драйвер `database`, это означает, что повторная проверка того же флага функции в рамках одного запроса не приведет к дополнительным запросам к базе данных. Это также гарантирует, что функция имеет последовательный результат в течение всего запроса.

Если вам нужно вручную очистить кэш в памяти, вы можете использовать метод `flushCache`, предоставленный фасадом `Feature`:

    Feature::flushCache();

<a name="scope"></a>
## Области (Scope)

<a name="specifying-the-scope"></a>
### Указание области

Как уже обсуждалось, функции обычно проверяются относительно текущего аутентифицированного пользователя. Однако это может не всегда соответствовать вашим потребностям. Поэтому возможно указать область, в которой вы хотите проверить данную функцию, с помощью метода `for` фасада `Feature`:

```php
return Feature::for($user)->active('new-api')
        ? $this->resolveNewApiResponse($request)
        : $this->resolveLegacyApiResponse($request);
```

Конечно, возможности функций не ограничиваются «пользователями». Представьте, что вы создали новый способ выставления счетов, которую вы распространяете на целые команды, а не на отдельных пользователей.  Возможно, вы хотели бы, чтобы для более старых команды новый способ внедрялся бы медленнее, чем для новых команды. Ваше замыкание для определения функции может выглядеть примерно так:

```php
use App\Models\Team;
use Carbon\Carbon;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('billing-v2', function (Team $team) {
    if ($team->created_at->isAfter(new Carbon('1st Jan, 2023'))) {
        return true;
    }

    if ($team->created_at->isAfter(new Carbon('1st Jan, 2019'))) {
        return Lottery::odds(1 / 100);
    }

    return Lottery::odds(1 / 1000);
});
```

Вы заметите, что определенное нами замыкание не ожидает `User`, а вместо этого ожидает модель `Team`. Чтобы определить, активна ли эта функция для команды пользователя, вы должны передать команду в метод `for`, предоставленный фасадом `Feature`:

```php
if (Feature::for($user->team)->active('billing-v2')) {
    return redirect('/billing/v2');
}

// ...
```

<a name="default-scope"></a>
### Область по умолчанию

Также возможно настроить область по умолчанию, которую Pennant использует для проверки функций. Например, возможно, все ваши функции проверяются относительно команды текущего аутентифицированного пользователя, а не самого пользователя. Вместо того чтобы вызывать `Feature::for($user->team)` каждый раз при проверке функции, вы можете указать команду в качестве области по умолчанию. Обычно это делается в одном из сервис-провайдеров вашего приложения:

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\Auth;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Feature::resolveScopeUsing(fn ($driver) => Auth::user()?->team);

        // ...
    }
}
```

Теперь, если область не указана явно с помощью метода `for`, проверка функции будет использовать команду текущего аутентифицированного пользователя в качестве области по умолчанию:

```php
Feature::active('billing-v2');

// Теперь эквивалентно...

Feature::for($user->team)->active('billing-v2');
```

<a name="nullable-scope"></a>
### Область с возможным значением NULL

Если область, которую вы передаете при проверке функции, равна `null`, а определение функции не поддерживает `null` с помощью nullable типа или не включает `null` в объединенный тип, Pennant автоматически вернет `false` в качестве значения результата функции.

Таким образом, если область, которую вы передаете функции, может быть `null`, и вы хотите, чтобы вызывался резольвер значения функции, вы должны учесть это в определении вашей функции. Область `null` может возникнуть, если вы проверяете функцию в команде Artisan, очередной задаче или в маршруте без аутентификации. Поскольку в этих контекстах обычно нет аутентифицированного пользователя, область по умолчанию будет `null`.

Если вы не всегда [явно указываете область вашей функции](#specifying-the-scope), то убедитесь, что тип области является "nullable" и обрабатывайте значение области `null` в логике определения вашей функции:

```php
use App\Models\User;
use Illuminate\Support\Lottery;
use Laravel\Pennant\Feature;

Feature::define('new-api', fn (User $user) => match (true) {// [tl! remove]
Feature::define('new-api', fn (User|null $user) => match (true) {// [tl! add]
    $user === null => true,// [tl! add]
    $user->isInternalTeamMember() => true,
    $user->isHighTrafficCustomer() => false,
    default => Lottery::odds(1 / 100),
});
```

<a name="identifying-scope"></a>
### Определение области

Встроенные драйверы хранения Pennant, такие как `array` и `database`, знают, как правильно хранить идентификаторы области для всех типов данных PHP, а также для моделей Eloquent. Однако, если ваше приложение использует сторонний драйвер Pennant, этот драйвер может не знать, как правильно хранить идентификатор для модели Eloquent или других пользовательских типов в вашем приложении.

В связи с этим Pennant позволяет вам форматировать значения области для хранения, реализовав контракт `FeatureScopeable` на объектах в вашем приложении, которые используются в качестве областей Pennant.

Например, представьте, что вы используете два разных драйвера функций в одном приложении: встроенный драйвер `database` и сторонний драйвер "Flag Rocket". Драйвер "Flag Rocket" не знает, как правильно хранить модель Eloquent. Вместо этого он требует экземпляр `FlagRocketUser`. Реализовав метод `toFeatureIdentifier`, определенный в контракте `FeatureScopeable`, мы можем настраивать значение области, которое будет предоставлено каждому драйверу, используемому в нашем приложении:

```php
<?php

namespace App\Models;

use FlagRocket\FlagRocketUser;
use Illuminate\Database\Eloquent\Model;
use Laravel\Pennant\Contracts\FeatureScopeable;

class User extends Model implements FeatureScopeable
{
    /**
     * Приведите объект к идентификатору области функций для данного драйвера.
     */
    public function toFeatureIdentifier(string $driver): mixed
    {
        return match($driver) {
            'database' => $this,
            'flag-rocket' => FlagRocketUser::fromId($this->flag_rocket_id),
        };
    }
}
```

<a name="serializing-scope"></a>
### Сериализация области

По умолчанию Pennant будет использовать полное имя класса при сохранении функции, связанной с моделью Eloquent. Если вы уже используете [карту полиморфных типов Eloquent](/docs/{{version}}/eloquent-relationships#custom-polymorphic-types), вы можете выбрать использование карты полиморфных типов также и в Pennant, чтобы отделить сохраненную функцию от структуры вашего приложения.

Для этого, после определения вашей карты полиморфных типов Eloquent в сервис-провайдере, вы можете вызвать метод `useMorphMap` фасада `Feature`:

```php
use Illuminate\Database\Eloquent\Relations\Relation;
use Laravel\Pennant\Feature;

Relation::enforceMorphMap([
    'post' => 'App\Models\Post',
    'video' => 'App\Models\Video',
]);

Feature::useMorphMap();
```

<a name="rich-feature-values"></a>
## Расширенные значения функций

До сих пор мы в основном показывали функции как бинарное состояние, что означает, что они либо "активны", либо "неактивны", но Pennant также позволяет вам хранить также и расширенные значения.

Например, представьте, что вы тестируете три новых цвета для кнопки "Купить сейчас" в вашем приложении. Вместо того, чтобы возвращать `true` или `false` из определения функции, вы можете вернуть строку:

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn (User $user) => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

Вы можете получить значение функции `purchase-button`, используя метод `value`:

```php
$color = Feature::value('purchase-button');
```

Включенная в Pennant директива Blade также упрощает условное отображение контента на основе текущего значения функции:

```blade
@feature('purchase-button', 'blue-sapphire')
    <!-- 'blue-sapphire' активен -->
@elsefeature('purchase-button', 'seafoam-green')
    <!-- 'seafoam-green' активен -->
@elsefeature('purchase-button', 'tart-orange')
    <!-- 'tart-orange' активен -->
@endfeature
```

> [!NOTE]
> При использовании расширенных значений важно знать, что функция считается "активной", когда она имеет любое значение, отличное от `false`.

При вызове [условного метода `when`](#conditional-execution), расширенное значение функции будет предоставлено первому замыканию:

    Feature::when('purchase-button',
        fn ($color) => /* ... */,
        fn () => /* ... */,
    );

Точно так же, при вызове условного метода `unless`, расширенное значение функции будет предоставлено второму необязательному замыканию:

    Feature::unless('purchase-button',
        fn () => /* ... */,
        fn ($color) => /* ... */,
    );

<a name="retrieving-multiple-features"></a>
## Получение нескольких функций

Метод `values` позволяет получить несколько функций для заданной области:

```php
Feature::values(['billing-v2', 'purchase-button']);

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
// ]
```

Или вы можете использовать метод `all`, чтобы получить значения всех определенных функций для заданной области:

```php
Feature::all();

// [
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

Однако функции на основе классов регистрируются динамически и неизвестны Pennant до тех пор, пока они явно не будут проверены. Это означает, что функции вашего приложения на основе классов могут не появиться в результатах, возвращаемых методом `all`, если они еще не были проверены в текущем запросе.

Если вы хотите гарантировать, что классы функций всегда будут включены при использовании метода `all`, вы можете использовать возможности обнаружения функций Pennant. Чтобы начать использование, вызовите метод `discover` в одном из сервис-провайдеров вашего приложения:

    <?php

    namespace App\Providers;

    use Illuminate\Support\ServiceProvider;
    use Laravel\Pennant\Feature;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Загрузка любых сервисов приложения.
         */
        public function boot(): void
        {
            Feature::discover();

            // ...
        }
    }

Метод `discover` зарегистрирует все классы функций в каталоге `app/Features` вашего приложения. Метод `all` теперь будет включать эти классы в свои результаты, независимо от того, были ли они проверены в текущем запросе:

```php
Feature::all();

// [
//     'App\Features\NewApi' => true,
//     'billing-v2' => false,
//     'purchase-button' => 'blue-sapphire',
//     'site-redesign' => true,
// ]
```

<a name="eager-loading"></a>
## Нетерпеливая загрузка

Хотя Pennant хранит в памяти кеш всех разрешенных функций для одного запроса, все же можно столкнуться с проблемами производительности. Чтобы облегчить эту проблему, Pennant предлагает возможность нетерпеливой загрузки значений функций.

Для иллюстрации этого представьте, что мы проверяем внутри цикла, активна ли функция:

```php
use Laravel\Pennant\Feature;

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

Предположим, что мы используем драйвер `database`. Этот код выполнит в цикле запрос к базе данных для каждого пользователя, что может привести к выполнению сотен запросов. Однако, используя метод `load` Pennant, мы можем устранить этот потенциально узкий момент производительности, предварительно загрузив значения функций для коллекции пользователей или областей:

```php
Feature::for($users)->load(['notifications-beta']);

foreach ($users as $user) {
    if (Feature::for($user)->active('notifications-beta')) {
        $user->notify(new RegistrationSuccess);
    }
}
```

Чтобы загрузить значения функций только в том случае, если они еще не были загружены, вы можете использовать метод `loadMissing`:

```php
Feature::for($users)->loadMissing([
    'new-api',
    'purchase-button',
    'notifications-beta',
]);
```

Вы можете загрузить все определенные функции, используя метод loadAll:

```php
Feature::for($users)->loadAll();
```

<a name="updating-values"></a>
## Обновление значений

Когда значение функции определяется впервые, базовый драйвер сохраняет результат в хранилище. Это часто необходимо для обеспечения единообразия поведения ваших пользователей во всех запросах. Однако иногда вам может потребоваться вручную обновить сохраненное значение функции.

Для этого вы можете использовать методы `activate` и `deactivate` для включения или выключения функции:

```php
use Laravel\Pennant\Feature;

// Активируем функцию для области действия по умолчанию...
Feature::activate('new-api');

// Деактивируем функцию для данной области действия...
Feature::for($user->team)->deactivate('billing-v2');
```

Также можно вручную установить расширенное значение для функции, предоставив второй аргумент методу `activate`:

```php
Feature::activate('purchase-button', 'seafoam-green');
```

Чтобы "забыть" сохраненное значение для функции, вы можете использовать метод forget. Когда функция будет проверена снова, Pennant будет разрешать значение функции из ее определения:

```php
Feature::forget('purchase-button');
```

<a name="bulk-updates"></a>
### Массовые обновления

Чтобы обновить сохраненные значения функций массово, вы можете использовать методы `activateForEveryone` и `deactivateForEveryone`.

Например, представьте, что теперь вы уверены в стабильности функции `new-api` и выбрали лучший цвет для кнопки `'purchase-button'` в вашем процессе оформления заказа - вы можете обновить сохраненное значение для всех пользователей соответственно:

```php
use Laravel\Pennant\Feature;

Feature::activateForEveryone('new-api');

Feature::activateForEveryone('purchase-button', 'seafoam-green');
```

В качестве альтернативы, вы можете деактивировать функцию для всех пользователей:

```php
Feature::deactivateForEveryone('new-api');
```

> [!NOTE]
> Это обновит только сохраненные значения разрешенных функций, которые были сохранены драйвером хранения Pennant. Вам также нужно будет обновить определение функции в вашем приложении.

<a name="purging-features"></a>
### Очистка функций

Иногда может быть полезно полностью удалить функцию из хранилища. Это обычно необходимо, если вы удалили функцию из своего приложения или внесли изменения в определение функции, которые вы хотели бы применить ко всем пользователям.

Вы можете удалить все сохраненные значения для функции, используя метод `purge`:

```php
// Очистка одной функции...
Feature::purge('new-api');

// Очистка нескольких функций...
Feature::purge(['new-api', 'purchase-button']);
```

Если вы хотите удалить _все_ функции из хранилища, вы можете вызвать метод `purge` без аргументов:

```php
Feature::purge();
```

Поскольку очищение функций может быть полезным в рамках процесса развёртывания вашего приложения, в Pennant имеется команда Artisan `pennant:purge`, которая будет удалять предоставленные функции из хранилища:

```sh
php artisan pennant:purge new-api

php artisan pennant:purge new-api purchase-button
```

Также можно очистить все функции _за исключением_ тех, что перечислены в определенном списке функций. Например, предположим, что вы хотите удалить все функции, кроме значений для "new-api" и "purchase-button", сохраненных в хранилище. Для этого передайте имена этих функций в опцию `--except`:

```sh
php artisan pennant:purge --except=new-api --except=purchase-button
```

Кроме того, для удобства команда `pennant:purge` также поддерживает флаг `--except-registered`. Этот флаг указывает, что нужно удалить все функции, кроме тех, которые явно зарегистрированы в сервис провайдере:

```sh
php artisan pennant:purge --except-registered
```

<a name="testing"></a>
## Тестирование

При тестировании кода, который взаимодействует с флагами функций, самым простым способом контролировать возвращаемое значение флага функции в ваших тестах является простое переопределение функции. Например, представьте, что у вас есть следующая функция, определенная в одном из сервис-провайдеров вашего приложения:

```php
use Illuminate\Support\Arr;
use Laravel\Pennant\Feature;

Feature::define('purchase-button', fn () => Arr::random([
    'blue-sapphire',
    'seafoam-green',
    'tart-orange',
]));
```

Чтобы изменить возвращаемое значение функции в ваших тестах, вы можете переопределить функцию в начале теста. Следующий тест всегда будет успешным, даже если реализация `Arr::random()` все еще присутствует в сервис-провайдере:

```php tab=Pest
use Laravel\Pennant\Feature;

test('он может контролировать значения функций', function () {
    Feature::define('purchase-button', 'seafoam-green');

    expect(Feature::value('purchase-button'))->toBe('seafoam-green');
});
```

```php tab=PHPUnit
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define('purchase-button', 'seafoam-green');

    $this->assertSame('seafoam-green', Feature::value('purchase-button'));
}
```

Тот же подход можно использовать и для функций на основе классов:

```php tab=Pest
use Laravel\Pennant\Feature;

test('он может контролировать значения функций', function () {
    Feature::define(NewApi::class, true);

    expect(Feature::value(NewApi::class))->toBeTrue();
});
```

```php tab=PHPUnit
use App\Features\NewApi;
use Laravel\Pennant\Feature;

public function test_it_can_control_feature_values()
{
    Feature::define(NewApi::class, true);

    $this->assertTrue(Feature::value(NewApi::class));
}
```

Если ваша функция возвращает экземпляр `Lottery`, доступно несколько полезных [вспомогательных инструментов для тестирования](/docs/{{version}}/helpers#testing-lotteries).

<a name="store-configuration"></a>
#### Конфигурация хранилища

Вы можете настроить хранилище, которое Pennant будет использовать во время тестирования, определив переменную окружения `PENNANT_STORE` в файле `phpunit.xml` вашего приложения:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit colors="true">
    <!-- ... -->
    <php>
        <env name="PENNANT_STORE" value="array"/>
        <!-- ... -->
    </php>
</phpunit>
```

<a name="adding-custom-pennant-drivers"></a>
## Добавление пользовательских драйверов Pennant

<a name="implementing-the-driver"></a>
#### Реализация драйвера

Если ни один из существующих драйверов хранения Pennant не подходит для вашего приложения, вы можете написать собственный драйвер хранения. Ваш пользовательский драйвер должен реализовать интерфейс `Laravel\Pennant\Contracts\Driver`:

```php
<?php

namespace App\Extensions;

use Laravel\Pennant\Contracts\Driver;

class RedisFeatureDriver implements Driver
{
    public function define(string $feature, callable $resolver): void {}
    public function defined(): array {}
    public function getAll(array $features): array {}
    public function get(string $feature, mixed $scope): mixed {}
    public function set(string $feature, mixed $scope, mixed $value): void {}
    public function setForAllScopes(string $feature, mixed $value): void {}
    public function delete(string $feature, mixed $scope): void {}
    public function purge(array|null $features): void {}
}
```

Теперь нам просто нужно реализовать каждый из этих методов, используя соединение с Redis. Для примера того, как реализовать каждый из этих методов, посмотрите на `Laravel\Pennant\Drivers\DatabaseDriver` в [исходном коде Pennant](https://github.com/laravel/pennant/blob/1.x/src/Drivers/DatabaseDriver.php).

> [!NOTE]
> Laravel не поставляется с каталогом для размещения ваших расширений. Вы можете разместить их где угодно. В этом примере мы создали каталог `Extensions`, чтобы разместить `RedisFeatureDriver`.

<a name="registering-the-driver"></a>
#### Регистрация драйвера

После того как ваш драйвер был реализован, вы готовы зарегистрировать его в Laravel. Чтобы добавить дополнительные драйверы в Pennant, вы можете использовать метод `extend`, предоставленный фасадом `Feature`. Вы должны вызвать метод `extend` из метода `boot` одного из [сервис-провайдеров](/docs/{{version}}/providers) вашего приложения:

```php
<?php

namespace App\Providers;

use App\Extensions\RedisFeatureDriver;
use Illuminate\Contracts\Foundation\Application;
use Illuminate\Support\ServiceProvider;
use Laravel\Pennant\Feature;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Зарегистрируйте любые службы приложения.
     */
    public function register(): void
    {
        // ...
    }

    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Feature::extend('redis', function (Application $app) {
            return new RedisFeatureDriver($app->make('redis'), $app->make('events'), []);
        });
    }
}
```

После регистрации драйвера вы можете использовать драйвер `redis` в файле конфигурации `config/pennant.php` вашего приложения:

    'stores' => [

        'redis' => [
            'driver' => 'redis',
            'connection' => null,
        ],

        // ...

    ],

<a name="defining-features-externally"></a>
### Внешнее определение объектов

Если ваш драйвер является оболочкой сторонней платформы с флагами функций, вы, скорее всего, будете определять функции на платформе, а не использовать метод Pennant `Feature::define`. В этом случае ваш собственный драйвер должен также реализовать интерфейс `Laravel\Pennant\Contracts\DefinesFeaturesExternally`:

```php
<?php

namespace App\Extensions;

use Laravel\Pennant\Contracts\Driver;
use Laravel\Pennant\Contracts\DefinesFeaturesExternally;

class FeatureFlagServiceDriver implements Driver, DefinesFeaturesExternally
{
    /**
     * Получение функции, определенные для данной области.
     */
    public function definedFeaturesForScope(mixed $scope): array {}

    /* ... */
}
```

Метод `definedFeaturesForScope` должен возвращать список имен функций, определенных для предоставленной области.

<a name="events"></a>
## События

Pennant отправляет различные события, которые могут быть полезны при отслеживании флагов функций в вашем приложении.

### `Laravel\Pennant\Events\FeatureRetrieved`

Это событие отправляется всякий раз, когда [проверяется функция](#checking-features). Это событие может быть полезно для создания и отслеживания показателей использования флага функции во всем приложении.

### `Laravel\Pennant\Events\FeatureResolved`

Это событие отправляется при первом разрешении значения функции для определенной области.

### `Laravel\Pennant\Events\UnknownFeatureResolved`

Это событие отправляется при первом разрешении неизвестной функции для определенной области. Прослушивание этого события может быть полезно, если вы намеревались удалить флаг функции, но случайно оставили на него случайные ссылки во всем приложении:

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnknownFeatureResolved;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Event::listen(function (UnknownFeatureResolved $event) {
            Log::error("Решение неизвестной функции [{$event->feature}].");
        });
    }
}
```

### `Laravel\Pennant\Events\DynamicallyRegisteringFeatureClass`

Это событие отправляется, когда [функция на основе класса](#class-based-features) динамически проверяется в первый раз во время запроса.

### `Laravel\Pennant\Events\UnexpectedNullScopeEncountered`

Это событие отправляется, когда область действия `null` передается определению функции, которая [не поддерживает null](#nullable-scope).

Эта ситуация корректно обрабатывается, и функция возвращает `false`. Однако, если вы хотите отказаться от корректного поведения этой функции по умолчанию, вы можете зарегистрировать прослушиватель этого события в методе `boot` вашего приложения `AppServiceProvider`:

```php
use Illuminate\Support\Facades\Log;
use Laravel\Pennant\Events\UnexpectedNullScopeEncountered;

/**
 * Загрузка любых сервисов приложения.
 */
public function boot(): void
{
    Event::listen(UnexpectedNullScopeEncountered::class, fn () => abort(500));
}

```

### `Laravel\Pennant\Events\FeatureUpdated`

Это событие отправляется при обновлении функции для области, обычно путем вызова `activate` или `deactivate`.

### `Laravel\Pennant\Events\FeatureUpdatedForAllScopes`

Это событие отправляется при обновлении функции для всех областей, обычно путем вызова `activateForEveryone` или `deactivateForEveryone`.

### `Laravel\Pennant\Events\FeatureDeleted`

Это событие отправляется при удалении функции для области, обычно путем вызова `forget`.

### `Laravel\Pennant\Events\FeaturesPurged`

Это событие отправляется при очистке определенных функций.

### `Laravel\Pennant\Events\AllFeaturesPurged`

Это событие отправляется при очистке всех функций.
