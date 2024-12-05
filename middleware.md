---
git: 85faadde4f004298773a1e0e8f0d53ae47430c77
---

# Посредники (middleware)

<a name="introduction"></a>
## Введение

Посредник обеспечивает удобный механизм для проверки и фильтрации HTTP-запросов, поступающих в ваше приложение. Например, в Laravel уже содержится посредник, проверяющий аутентификацию пользователя вашего приложения. Если пользователь не аутентифицирован, то посредник перенаправит пользователя на экран входа в ваше приложение. Однако, если пользователь аутентифицирован, то посредник позволит запросу продолжить работу в приложении.

Посредник может быть написан для выполнения различных задач помимо аутентификации. Например, посредник для ведения журналов может регистрировать все входящие запросы к вашему приложению. В Laravel включено множество посредников, включая псредники для аутентификации и защиты CSRF; однако все определяемые пользователями посредники обычно находится в каталоге `app/Http/Middleware` вашего приложения.

<a name="defining-middleware"></a>
## Определение посредника

Чтобы создать нового посредника, используйте команду `make:middleware` [Artisan](artisan):

```shell
php artisan make:middleware EnsureTokenIsValid
```

Эта команда поместит новый класс посредника в каталог `app/Http/Middleware` вашего приложения. В этом посреднике мы будем разрешать доступ к маршруту только в том случае, если значение входящего `token` соответствует указанному. В противном случае мы перенаправим пользователя по маршруту `/home`:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class EnsureTokenIsValid
    {
        /**
         * Обработка входящего запроса.
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            if ($request->input('token') !== 'my-secret-token') {
                return redirect('/home');
            }

            return $next($request);
        }
    }

Как видите, если переданный `token` не совпадает с нашим секретным токеном, то посредник вернет клиенту HTTP-перенаправление; в противном случае запрос будет передан в приложение. Чтобы передать запрос дальше в приложение (позволяя «пройти» посредника), вы должны вызвать замыкание `$next` с параметром `$request`.

Лучше всего представить себе посредников как серию «слоев» для HTTP-запроса, которые необходимо пройти, прежде чем запрос попадет в ваше приложение. Каждый слой может рассмотреть запрос и даже полностью отклонить его.

> [!NOTE]
> Все посредники извлекаются из [контейнера служб](/docs/{{version}}/container), поэтому вы можете объявить необходимые вам зависимости в конструкторе посредника.

<a name="middleware-and-responses"></a>
#### Посредники и ответы

Конечно, посредник может выполнять задачи до или после передачи запроса в приложение. Например, следующий посредник будет выполнять некоторую задачу **до** того, как запрос будет обработан приложением:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class BeforeMiddleware
    {
        public function handle(Request $request, Closure $next): Response
        {
            // Выполнить действие

            return $next($request);
        }
    }

Однако, этот посредник будет выполнять свою задачу **после** обработки входящего запроса приложением:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class AfterMiddleware
    {
        public function handle(Request $request, Closure $next): Response
        {
            $response = $next($request);

            // Выполнить действие

            return $response;
        }
    }

<a name="registering-middleware"></a>
## Регистрация посредника

<a name="global-middleware"></a>
### Глобальный стек HTTP-посредников

Если вы хотите, чтобы посредник запускался при каждом HTTP-запросе к вашему приложению, вы можете добавить его в глобальный стек посредников в файле `bootstrap/app.php` вашего приложения:

    use App\Http\Middleware\EnsureTokenIsValid;

    ->withMiddleware(function (Middleware $middleware) {
         $middleware->append(EnsureTokenIsValid::class);
    })

Объект `$middleware`, предоставляемый замыканию `withMiddleware`, является экземпляром `Illuminate\Foundation\Configuration\Middleware` и отвечает за управление посредником, назначенным маршрутам вашего приложения. Метод `append` добавляет посредника в конец глобального списка посредников. Если вы хотите добавить посредника в начало списка, вам следует использовать метод `prepend`.

<a name="manually-managing-laravels-default-global-middleware"></a>
#### Ручное управление глобальным стеком посредников

Если вы хотите управлять глобальным стеком посредников Laravel вручную, вы можете предоставить глобальный стек посредников Laravel по умолчанию для метода `use`. Затем вы можете при необходимости настроить стек посредников по умолчанию:

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->use([
            \Illuminate\Foundation\Http\Middleware\InvokeDeferredCallbacks::class,
            // \Illuminate\Http\Middleware\TrustHosts::class,
            \Illuminate\Http\Middleware\TrustProxies::class,
            \Illuminate\Http\Middleware\HandleCors::class,
            \Illuminate\Foundation\Http\Middleware\PreventRequestsDuringMaintenance::class,
            \Illuminate\Http\Middleware\ValidatePostSize::class,
            \Illuminate\Foundation\Http\Middleware\TrimStrings::class,
            \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
        ]);
    })

<a name="assigning-middleware-to-routes"></a>
### Назначение посредников маршрутам

Если вы хотите назначить посредника (middleware) для определенных маршрутов, вы можете использовать метод `middleware` при определении маршрута:

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::get('/profile', function () {
        // ...
    })->middleware(EnsureTokenIsValid::class);

Вы также можете назначить несколько middleware для маршрута, передав массив имен в метод `middleware`:

    Route::get('/', function () {
        // ...
    })->middleware([First::class, Second::class]);

<a name="excluding-middleware"></a>
#### Исключение посредников

При назначении посредника группе маршрутов, иногда может потребоваться запретить применение посредника к одному из маршрутов в группе. Вы можете сделать это с помощью метода `withoutMiddleware`:

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::middleware([EnsureTokenIsValid::class])->group(function () {
        Route::get('/', function () {
            // ...
        });

        Route::get('/profile', function () {
            // ...
        })->withoutMiddleware([EnsureTokenIsValid::class]);
    });

Вы также можете исключить данный набор посредников из всей [группы маршрутов](/docs/{{version}}/routing#route-groups):

    use App\Http\Middleware\EnsureTokenIsValid;

    Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
        Route::get('/profile', function () {
            // ...
        });
    });

Метод `withoutMiddleware` удаляет только посредника маршрутизации и не применим к [глобальному посреднику](#global-middleware).

<a name="middleware-groups"></a>
### Группы посредников

По желанию можно сгруппировать несколько посредников под одним ключом, чтобы упростить их назначение маршрутам. Вы можете сделать это, используя метод `appendToGroup` в файле `bootstrap/app.php` вашего приложения:

    use App\Http\Middleware\First;
    use App\Http\Middleware\Second;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->appendToGroup('group-name', [
            First::class,
            Second::class,
        ]);

        $middleware->prependToGroup('group-name', [
            First::class,
            Second::class,
        ]);
    })

Группы посредников могут быть назначены маршрутам и действиям контроллера, используя тот же синтаксис, что и с индивидуальным посредником:

    Route::get('/', function () {
        // ...
    })->middleware('group-name');

    Route::middleware(['group-name'])->group(function () {
        // ...
    });

<a name="laravels-default-middleware-groups"></a>
#### Группы посредников Laravel по умолчанию

Laravel включает в себя предопределенные группы посредников `web` и `api`, которые содержат общие посредники, которое вы, возможно, захотите применить к своим веб и API маршрутам. Помните, что Laravel автоматически применяет эти группы посредников к соответствующим файлам `routes/web.php` и `routes/api.php`:

| Группа посредников `web`                                  |
| --------------------------------------------------------- |
| `Illuminate\Cookie\Middleware\EncryptCookies`             |
| `Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse` |
| `Illuminate\Session\Middleware\StartSession`              |
| `Illuminate\View\Middleware\ShareErrorsFromSession`       |
| `Illuminate\Foundation\Http\Middleware\ValidateCsrfToken` |
| `Illuminate\Routing\Middleware\SubstituteBindings`        |

| Группа посредников `api`                           |
| -------------------------------------------------- |
| `Illuminate\Routing\Middleware\SubstituteBindings` |

Если вы хотите добавить или добавить посредника к этим группам, вы можете использовать методы `web` и `api` в файле `bootstrap/app.php` вашего приложения. Методы `web` и `api` являются удобной альтернативой методу `appendToGroup`:

    use App\Http\Middleware\EnsureTokenIsValid;
    use App\Http\Middleware\EnsureUserIsSubscribed;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->web(append: [
            EnsureUserIsSubscribed::class,
        ]);

        $middleware->api(prepend: [
            EnsureTokenIsValid::class,
        ]);
    })

Вы даже можете заменить одну из записей группы посредников Laravel по умолчанию на собственного посредника:

    use App\Http\Middleware\StartCustomSession;
    use Illuminate\Session\Middleware\StartSession;

    $middleware->web(replace: [
        StartSession::class => StartCustomSession::class,
    ]);

Или вы можете полностью удалить посредника:

    $middleware->web(remove: [
        StartSession::class,
    ]);

<a name="manually-managing-laravels-default-middleware-groups"></a>
#### Ручное управление группами посредников Laravel по умолчанию

Если вы хотите вручную управлять всеми посредниками в группах посредников Laravel по умолчанию `web` и `api`, вы можете полностью переопределить эти группы. В приведенном ниже примере будут определены группы посредников `web` и `api` с их посредниками по умолчанию, что позволит вам настроить их по мере необходимости:

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->group('web', [
            \Illuminate\Cookie\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
            // \Illuminate\Session\Middleware\AuthenticateSession::class,
        ]);

        $middleware->group('api', [
            // \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            // 'throttle:api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ]);
    })

> [!NOTE]
> По умолчанию группы посредников `web` и `api` автоматически применяются к соответствующим файлам `routes/web.php` и `routes/api.php` вашего приложения с помощью файла `bootstrap/app.php`.

<a name="middleware-aliases"></a>
### Псевдонимы посредников

Вы можете назначить псевдонимы посредникам в файле `bootstrap/app.php` вашего приложения. Псевдонимы посредников позволяют определить короткий псевдоним для данного класса посредника, что может быть особенно полезно для посредника с длинными именами классов:

    use App\Http\Middleware\EnsureUserIsSubscribed;

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'subscribed' => EnsureUserIsSubscribed::class
        ]);
    })

После того как псевдоним посредника определен в файле `bootstrap/app.php` вашего приложения, вы можете использовать его при назначении посредника маршрутам:

    Route::get('/profile', function () {
        // ...
    })->middleware('subscribed');

Для удобства некоторые встроенные посредники Laravel по умолчанию имеют псевдонимы. Например, посредник `auth` является псевдонимом посредника `Illuminate\Auth\Middleware\Authenticate`. Ниже приведен список псевдонимов посредников по умолчанию:

| Псевдоним          | Посредник                                                                                                      |
| ------------------ | -------------------------------------------------------------------------------------------------------------- |
| `auth`             | `Illuminate\Auth\Middleware\Authenticate`                                                                      |
| `auth.basic`       | `Illuminate\Auth\Middleware\AuthenticateWithBasicAuth`                                                         |
| `auth.session`     | `Illuminate\Session\Middleware\AuthenticateSession`                                                            |
| `cache.headers`    | `Illuminate\Http\Middleware\SetCacheHeaders`                                                                   |
| `can`              | `Illuminate\Auth\Middleware\Authorize`                                                                         |
| `guest`            | `Illuminate\Auth\Middleware\RedirectIfAuthenticated`                                                           |
| `password.confirm` | `Illuminate\Auth\Middleware\RequirePassword`                                                                   |
| `precognitive`     | `Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests`                                             |
| `signed`           | `Illuminate\Routing\Middleware\ValidateSignature`                                                              |
| `subscribed`       | `\Spark\Http\Middleware\VerifyBillableIsSubscribed`                                                            |
| `throttle`         | `Illuminate\Routing\Middleware\ThrottleRequests` или `Illuminate\Routing\Middleware\ThrottleRequestsWithRedis` |
| `verified`         | `Illuminate\Auth\Middleware\EnsureEmailIsVerified`                                                             |

<a name="sorting-middleware"></a>
### Сортировка посредников

В редких случаях вам может потребоваться, чтобы ваши посредники выполнялись в определенном порядке, но вы не можете контролировать их порядок, когда они назначены маршруту. В этом случае вы можете указать приоритет посредников, используя метод `priority` в файле `bootstrap/app.php` вашего приложения:

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->priority([
            \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,
            \Illuminate\Cookie\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            \Illuminate\Routing\Middleware\ThrottleRequests::class,
            \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
            \Illuminate\Contracts\Auth\Middleware\AuthenticatesRequests::class,
            \Illuminate\Auth\Middleware\Authorize::class,
        ]);
    })

<a name="middleware-parameters"></a>
## Параметры посредника

Посредник также может получать дополнительные параметры. Например, если вашему приложению необходимо проверить, что аутентифицированный пользователь имеет конкретную «роль» перед выполнением им конкретного действия, то вы можете создать посредника, например, `EnsureUserHasRole`, который получит имя роли в качестве дополнительного аргумента.

Дополнительные параметры посредника будут переданы после аргумента `$next`:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class EnsureUserHasRole
    {
        /**
         * Обработка входящего запроса.
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next, string $role): Response
        {
            if (! $request->user()->hasRole($role)) {
                // Перенаправление...
            }

            return $next($request);
        }

    }

Параметры посредника можно указать при определении маршрута, разделив имя посредника и параметры символом `:`.

    use App\Http\Middleware\EnsureUserHasRole;

    Route::put('/post/{id}', function (string $id) {
        // ...
    })->middleware(EnsureUserHasRole::class.':editor');

Несколько параметров следует разделять запятыми:

    Route::put('/post/{id}', function (string $id) {
        // ...
    })->middleware(EnsureUserHasRole::class.':editor,publisher');

<a name="terminable-middleware"></a>
## Завершающий посредник

Иногда посреднику может потребоваться выполнить некоторую работу после отправки HTTP-ответа в браузер. Если вы определите метод `terminate` в своем посреднике и при условии, что ваш веб-сервер использует FastCGI, то метод `terminate` будет автоматически вызван после отправки ответа в браузер:

    <?php

    namespace Illuminate\Session\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class TerminatingMiddleware
    {
        /**
         * Обработка входящего запроса.
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            return $next($request);
        }

        /**
         * Обработать задачи после отправки ответа в браузер.
         */
        public function terminate(Request $request, Response $response): void
        {
            // ...
        }
    }

Метод `terminate` должен получать и запрос, и ответ. После того как вы определили завершающий посредник, вы должны добавить его в список маршрутов или глобальный стек посредников в файле `bootstrap/app.php` вашего приложения.

При вызове метода `terminate` посредника, Laravel извлечет новый экземпляр посредника из [контейнера служб](/docs/{{version}}/container). Если вы хотите использовать один и тот же экземпляр посредника при вызове методов `handle` и `terminate`, то зарегистрируйте посредника в контейнере, используя метод контейнера `singleton`. Обычно это должно быть сделано в методе `register` вашего `AppServiceProvider`:

    use App\Http\Middleware\TerminatingMiddleware;

    /**
     * Регистрация любых служб приложения.
     */
    public function register(): void
    {
        $this->app->singleton(TerminatingMiddleware::class);
    }
