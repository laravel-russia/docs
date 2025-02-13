---
git: d636b0efcb462b894e8f18dd73f1b72f37a74881
---

# Авторизация

<a name="introduction"></a>
## Введение

Помимо встроенных служб [аутентификации](authentication), Laravel также предлагает простой способ авторизации действий пользователя с конкретными ресурсами. Например, даже если пользователь аутентифицирован, то он может быть не авторизован для обновления или удаления определенных моделей Eloquent или записей базы данных вашего приложения. Функционал авторизации Laravel обеспечивает простой и организованный способ управления этими проверками авторизации.

Laravel предлагает два основных способа авторизации действий: [шлюзы](#gates) и [политики](#creating-policies). Думайте о шлюзах и политиках, как о маршрутах и контроллерах. Шлюзы обеспечивают простой подход к авторизации, основанный на замыкании, в то время как политики, также как контроллеры, группируют логику вокруг конкретной модели или ресурса. В этой документации мы сначала рассмотрим шлюзы, а затем политики.

Вам не нужно выбирать между использованием исключительно Gates (шлюзов) или исключительно Policies (политик) при создании приложения.  большинстве приложений, скорее всего, будет использоваться комбинация обоих, и это совершенно нормально! Gates наиболее подходят для действий, не связанных с какой-либо моделью или ресурсом, например, для просмотра панели управления администратора. В свою очередь, Policies следует использовать, когда вы хотите авторизовать действие для конкретной модели или ресурса.

<a name="gates"></a>
## Шлюзы (Gates)

<a name="writing-gates"></a>
### Написание шлюзов

> [!WARNING]
> Шлюзы – отличный способ изучить основы функционала авторизации Laravel; однако при создании надежных приложений Laravel, вам следует рассмотреть возможность использования [политик](#creating-policies) для организации ваших правил авторизации.

Шлюз – это просто замыкание, которое определяет, имеет ли пользователь право выполнять указанное действие. Обычно шлюзы определяются в методе `boot` класса `App\Providers\AppServiceProvider` с использованием фасада `Gate`. Шлюзы всегда получают экземпляр пользователя в качестве своего первого аргумента и могут получать дополнительные аргументы, например, модель Eloquent.

В этом примере мы определим шлюз, решающий, может ли пользователь обновить указанную модель `App\Models\Post`. Шлюз выполнит это, сравнив идентификатор пользователя с идентификатором `user_id` пользователя, создавшего пост:

    use App\Models\Post;
    use App\Models\User;
    use Illuminate\Support\Facades\Gate;

    /**
     * Запуск любых служб приложений.
     */
    public function boot(): void
    {
        Gate::define('update-post', function (User $user, Post $post) {
            return $user->id === $post->user_id;
        });
    }

Шлюзы также могут быть определены с использованием callback-массива:

    use App\Policies\PostPolicy;
    use Illuminate\Support\Facades\Gate;

    /**
     * Запуск любых служб приложений.
     */
    public function boot(): void
    {
        Gate::define('update-post', [PostPolicy::class, 'update']);
    }

<a name="authorizing-actions-via-gates"></a>
### Авторизация действий через шлюзы

Чтобы авторизовать действие с помощью шлюзов, вы должны использовать методы `allows` или `denies` фасада `Gate`. Обратите внимание, что вам не требуется передавать в эти методы аутентифицированного в данный момент пользователя. Laravel автоматически позаботится о передаче пользователя в замыкание шлюза. Обычно методы авторизации шлюза вызываются в контроллерах вашего приложения перед выполнением действия, требующего авторизации:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Models\Post;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Gate;

    class PostController extends Controller
    {
        /**
         * Обновить переданный пост.
         */
        public function update(Request $request, Post $post): RedirectResponse
        {
            if (! Gate::allows('update-post', $post)) {
                abort(403);
            }

            // Обновление поста...

            return redirect('/posts');
        }
    }

Если вы хотите определить, авторизован ли другой (не аутентифицированный в настоящий момент) пользователь для выполнения действия, то вы можете использовать метод `forUser` фасада `Gate`:

    if (Gate::forUser($user)->allows('update-post', $post)) {
        // Пользователь может обновить пост...
    }

    if (Gate::forUser($user)->denies('update-post', $post)) {
        // Пользователь не может обновить пост...
    }

Вы можете определить авторизацию нескольких действий одновременно, используя методы `any` или `none`:

    if (Gate::any(['update-post', 'delete-post'], $post)) {
        // Пользователь может обновить или удалить пост...
    }

    if (Gate::none(['update-post', 'delete-post'], $post)) {
        // Пользователь не может обновить или удалить пост...
    }

<a name="authorizing-or-throwing-exceptions"></a>
#### Авторизация или выброс исключений

Если вы хотите попытаться авторизовать действие и автоматически выдать исключение `Illuminate\Auth\Access\AuthorizationException`, если пользователю не разрешено выполнять данное действие, вы можете использовать метод `authorize` фасада `Gate`. Экземпляры `AuthorizationException` автоматически преобразуются Laravel в HTTP-ответ 403:

    Gate::authorize('update-post', $post);

    // Действие разрешено...

<a name="gates-supplying-additional-context"></a>
#### Предоставление дополнительного контекста шлюзам

Методы шлюза для авторизации полномочий (`allows`, `denies`, `check`, `any`, `none`, `authorize`, `can`, `cannot`) и [директивы авторизации Blade](#via-blade-templates) (`@can`, `@cannot`, `@canany`) могут получать массив в качестве второго аргумента. Эти элементы массива передаются в качестве параметров замыканию шлюза и могут использоваться как дополнительный контекст при принятии решений об авторизации:

    use App\Models\Category;
    use App\Models\User;
    use Illuminate\Support\Facades\Gate;

    Gate::define('create-post', function (User $user, Category $category, bool $pinned) {
        if (! $user->canPublishToGroup($category->group)) {
            return false;
        } elseif ($pinned && ! $user->canPinPosts()) {
            return false;
        }

        return true;
    });

    if (Gate::check('create-post', [$category, $pinned])) {
        // Пользователь может создать пост...
    }

<a name="gate-responses"></a>
### Ответы шлюза

До сих пор мы рассматривали шлюзы, возвращающие простые логические значения. По желанию можно вернуть более подробный ответ, содержащий также сообщение об ошибке. Для этого вы можете вернуть экземпляр `Illuminate\Auth\Access\Response` из вашего шлюза:

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
                    ? Response::allow()
                    : Response::deny('Вы должны быть администратором.');
    });

Даже когда вы возвращаете ответ авторизации из вашего шлюза, метод `Gate::allows` все равно будет возвращать простое логическое значение; однако вы можете использовать метод `Gate::inspect`, чтобы получить полный возвращенный шлюзом ответ авторизации:

    $response = Gate::inspect('edit-settings');

    if ($response->allowed()) {
        // Действие разрешено...
    } else {
        echo $response->message();
    }

При использовании метода `Gate::authorize`, генерирующего исключение `AuthorizationException` для неавторизованного действия, сообщение об ошибке из ответа авторизации будет передано в HTTP-ответ:

    Gate::authorize('edit-settings');

    // Действие разрешено...

<a name="customizing-gate-response-status"></a>
#### Настройка статуса HTTP-ответа

Когда доступ к действию запрещен через Gate, возвращается HTTP-ответ с кодом `403`. Однако иногда может быть полезно возвращать другой HTTP-статус. Вы можете настроить код статуса HTTP, который возвращается при неудачной проверке авторизации, используя статический конструктор `denyWithStatus` в классе `Illuminate\Auth\Access\Response`:

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
                    ? Response::allow()
                    : Response::denyWithStatus(404);
    });

Поскольку скрытие ресурсов с помощью ответа `404` является общепринятым подходом в веб-приложениях, для удобства предлагается метод `denyAsNotFound`:

    use App\Models\User;
    use Illuminate\Auth\Access\Response;
    use Illuminate\Support\Facades\Gate;

    Gate::define('edit-settings', function (User $user) {
        return $user->isAdmin
                    ? Response::allow()
                    : Response::denyAsNotFound();
    });

<a name="intercepting-gate-checks"></a>
### Хуки шлюзов

Иногда бывает необходимо предоставить все полномочия конкретному пользователю. Вы можете использовать метод `before` для определения замыкания, которое выполняется перед всеми другими проверками авторизации:

    use App\Models\User;
    use Illuminate\Support\Facades\Gate;

    Gate::before(function (User $user, string $ability) {
        if ($user->isAdministrator()) {
            return true;
        }
    });

Если замыкание `before` возвращает результат, отличный от `null`, то этот результат и будет считаться результатом проверки авторизации.

Вы можете использовать метод `after` для определения замыкания, которое будет выполнено после всех других проверок авторизации:

    use App\Models\User;

    Gate::after(function (User $user, string $ability, bool|null $result, mixed $arguments) {
        if ($user->isAdministrator()) {
            return true;
        }
    });

Значения, возвращаемые замыканиями `after`, не будут переопределять результат проверки авторизации, если шлюз или политика не возвратят `null`.

<a name="inline-authorization"></a>
### Встроенная авторизация

Иногда вы можете захотеть определить, авторизован ли текущий аутентифицированный пользователь для выполнения данного действия без написания специального шлюза, соответствующего этому действию. Laravel позволяет вам выполнять эти типы «встроенных» проверок авторизации с помощью методов `Gate::allowIf` и `Gate::denyIf`. Встроенная авторизация не выполняет никаких определенных ["before" или "after" хуков авторизации](#intercepting-gate-checks):

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::allowIf(fn (User $user) => $user->isAdministrator());

Gate::denyIf(fn (User $user) => $user->banned());
```

Если действие не авторизовано или ни один пользователь в настоящее время не аутентифицирован, Laravel автоматически выдаст исключение `Illuminate\Auth\Access\AuthorizationException`. Экземпляры `AuthorizationException` автоматически преобразуются в HTTP-ответ 403 обработчиком исключений Laravel:

<a name="creating-policies"></a>
## Создание политик

<a name="generating-policies"></a>
### Генерация политик

Политики – это классы, которые организуют логику авторизации для конкретной модели или ресурса. Например, если ваше приложение является блогом, то у вас может быть модель `App\Models\Post` и соответствующая политика `App\Policies\PostPolicy` для авторизации действий пользователя, например, создание или обновление постов.

Чтобы сгенерировать новую политику, используйте команду `make:policy` [Artisan](artisan). Эта команда поместит новый класс политики в каталог `app/Policies` вашего приложения. Если этот каталог еще не существует, то Laravel предварительно создаст его:

```shell
php artisan make:policy PostPolicy
```

Команда `make:policy` сгенерирует пустой класс политики. Если вы хотите создать класс с заготовками методов политики, связанных с просмотром, созданием, обновлением и удалением ресурса, то вы можете указать параметр `--model` при выполнении команды:

```shell
php artisan make:policy PostPolicy --model=Post
```

<a name="registering-policies"></a>
### Регистрация политик

<a name="policy-discovery"></a>
#### Обнаружение политики

По умолчанию Laravel автоматически обнаруживает политики, если модель и политика соответствуют стандартным соглашениям об именах Laravel. В частности, политики должны находиться в каталоге `Policies`, расположенном в каталоге, содержащем ваши модели, или выше него. Так, например, модели могут быть размещены в каталоге `app/Models`, а политики — в каталоге `app/Policies`. В этой ситуации Laravel проверит наличие политик в `app/Models/Policies`, а затем в `app/Policies`. Кроме того, имя политики должно совпадать с названием модели и иметь суффикс `Policy`. Таким образом, модель `User` будет соответствовать классу политики `UserPolicy`.

Если вы хотите определить свою собственную логику обнаружения политики, вы можете зарегистрировать обратный вызов обнаружения собственной политики с помощью метода `Gate::guessPolicyNamesUsing`. Обычно этот метод следует вызывать из метода `boot` `AppServiceProvider` вашего приложения:

    use Illuminate\Support\Facades\Gate;

    Gate::guessPolicyNamesUsing(function (string $modelClass) {
        // Возвращаем имя класса политики для данной модели...
    });

<a name="manually-registering-policies"></a>
#### Регистрация политик вручную

Используя фасад `Gate`, вы можете вручную регистрировать политики и соответствующие им модели в методе `boot` `AppServiceProvider` вашего приложения:

    use App\Models\Order;
    use App\Policies\OrderPolicy;
    use Illuminate\Support\Facades\Gate;

    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Gate::policy(Order::class, OrderPolicy::class);
    }

<a name="writing-policies"></a>
## Написание политик

<a name="policy-methods"></a>
### Методы политики

После регистрации класса политики вы можете добавить методы для каждого из авторизуемых действий. Например, давайте определим метод `update` в нашем классе `PostPolicy`, который решает, может ли пользователь обновить указанный экземпляр поста.

Метод `update` получит в качестве аргументов экземпляры `User` и `Post` и должен вернуть `true` или `false`, которые будут указывать, авторизован ли пользователь обновлять указанный пост. Итак, в этом примере мы проверим, что идентификатор пользователя совпадает с `user_id` поста:

    <?php

    namespace App\Policies;

    use App\Models\Post;
    use App\Models\User;

    class PostPolicy
    {
        /**
         * Определить, может ли пользователь обновить пост.
         */
        public function update(User $user, Post $post): bool
        {
            return $user->id === $post->user_id;
        }
    }

Вы можете продолжить определение в политике необходимых методов дополнительных авторизуемых действий. Например, вы можете определить методы `view` или `delete` для авторизации различных действий, связанных с `Post`. Помните, что вы можете дать своим методам политики любые желаемые имена.

Если вы использовали опцию `--model` при создании своей политики через Artisan, то она уже будет содержать методы для следующих действий: `viewAny`, `view`, `create`, `update`, `delete`, `restore`, и `forceDelete`.

> [!NOTE]
> Все политики извлекаются через [контейнер служб](/docs/{{version}}/container) Laravel, что позволяет вам объявлять любые необходимые зависимости в конструкторе политики для их автоматического внедрения.

<a name="policy-responses"></a>
### Ответы политики

До сих пор мы рассматривали методы политики, возвращающие простые логические значения. По желанию можно вернуть более подробный ответ, содержащий также сообщение об ошибке. Для этого вы можете вернуть экземпляр `Illuminate\Auth\Access\Response` из вашего метода политики:

    use App\Models\Post;
    use App\Models\User;
    use Illuminate\Auth\Access\Response;

    /**
     * Определить, может ли пользователь обновить пост.
     */
    public function update(User $user, Post $post): Response
    {
        return $user->id === $post->user_id
                    ? Response::allow()
                    : Response::deny('You do not own this post.');
    }

При возврате ответа авторизации из вашей политики метод `Gate::allows` все равно будет возвращать простое логическое значение; однако вы можете использовать метод `Gate::inspect`, чтобы получить полный возвращенный шлюзом ответ авторизации:

    use Illuminate\Support\Facades\Gate;

    $response = Gate::inspect('update', $post);

    if ($response->allowed()) {
        // Действие разрешено...
    } else {
        echo $response->message();
    }

При использовании метода `Gate::authorize`, генерирующего исключение `AuthorizationException` для неавторизованного действия, сообщение об ошибке из ответа авторизации будет передано в HTTP-ответ:

    Gate::authorize('update', $post);

    // Действие разрешено...

<a name="customizing-policy-response-status"></a>
#### Настройка статуса HTTP-ответа

Когда действие запрещается методом политики, возвращается ответ HTTP со статусом `403`. Однако иногда может быть полезно вернуть другой статус HTTP. Вы можете настроить код статуса HTTP, возвращаемый при неудачной проверке авторизации, используя статический конструктор denyWithStatus в классе `Illuminate\Auth\Access\Response`:

    use App\Models\Post;
    use App\Models\User;
    use Illuminate\Auth\Access\Response;

    /**
     * Определяет, может ли данный пользователь обновить указанный пост.
     */
    public function update(User $user, Post $post): Response
    {
        return $user->id === $post->user_id
                    ? Response::allow()
                    : Response::denyWithStatus(404);
    }

Поскольку скрытие ресурсов с помощью ответа `404` является общепринятым подходом в веб-приложениях, для удобства предлагается метод `denyAsNotFound`:

    use App\Models\Post;
    use App\Models\User;
    use Illuminate\Auth\Access\Response;

    /**
     * Определяет, может ли данный пользователь обновить указанный пост.
     */
    public function update(User $user, Post $post): Response
    {
        return $user->id === $post->user_id
                    ? Response::allow()
                    : Response::denyAsNotFound();
    }

<a name="methods-without-models"></a>
### Методы политики без моделей

Некоторые методы политики получают только экземпляр аутентифицированного в данный момент пользователя. Эта ситуация наиболее распространена при авторизации действий `create`. Например, если вы создаете блог, то вы можете определить, имеет ли пользователь право вообще создавать какие-либо посты. В этих ситуациях ваш метод политики должен рассчитывать только на получение экземпляра пользователя:

    /**
     * Определить, может ли пользователь создать пост.
     */
    public function create(User $user): bool
    {
        return $user->role == 'writer';
    }

<a name="guest-users"></a>
### Гостевые пользователи

По умолчанию все шлюзы и политики автоматически возвращают `false`, если входящий HTTP-запрос был инициирован не аутентифицированным пользователем. Однако вы можете разрешить прохождение этих проверок авторизации к вашим шлюзам и политикам, пометив в аргументе метода объявленный тип `User` как [обнуляемый](https://www.php.net/manual/ru/language.types.declarations.php#language.types.declarations.nullable), путём добавления префикса в виде знака вопроса (`?`). Это означает, что значение может быть как объявленного типа `User`, так и быть равным `null`:

    <?php

    namespace App\Policies;

    use App\Models\Post;
    use App\Models\User;

    class PostPolicy
    {
        /**
         * Определить, может ли пользователь обновить пост.
         */
        public function update(?User $user, Post $post): bool
        {
            return $user?->id === $post->user_id;
        }
    }

<a name="policy-filters"></a>
### Фильтры политики

Для определенных пользователей вы можете разрешить все действия в рамках конкретной политики. Для этого определите в политике метод `before`. Метод `before` будет выполнен перед любыми другими методами в политике, что даст вам возможность авторизовать действие до фактического вызова предполагаемого метода политики. Этот функционал чаще всего используется для авторизации администраторов приложения на выполнение любых действий:

    use App\Models\User;

    /**
     * Выполнить предварительную авторизацию.
     */
    public function before(User $user, string $ability): bool|null
    {
        if ($user->isAdministrator()) {
            return true;
        }

        return null;
    }

Если вы хотите отклонить все проверки авторизации для определенного типа пользователей, вы можете вернуть `false` из метода `before`. Если возвращается `null`, то проверка авторизации перейдет к методу политики.

> [!WARNING]
> Метод `before` класса политики не будет вызываться, если класс не содержит метода с именем, совпадающим с именем проверяемого полномочия.

<a name="authorizing-actions-using-policies"></a>
## Авторизация действий с помощью политик

<a name="via-the-user-model"></a>
### Авторизация действий с помощью политик через модель User

Модель `App\Models\User` приложения Laravel включает два полезных метода авторизации действий: `can` и `cannot`. Методы `can` и `cannot` получают имя действия, которое вы хотите авторизовать, и соответствующую модель. Например, давайте определим, авторизован ли пользователь для обновления переданной модели `App\Models\Post`. Обычно это делается в методе контроллера:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Models\Post;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class PostController extends Controller
    {
        /**
         * Обновить переданный пост.
         */
        public function update(Request $request, Post $post): RedirectResponse
        {
            if ($request->user()->cannot('update', $post)) {
                abort(403);
            }

            // Обновление поста...

            return redirect('/posts');
        }
    }

Если для данной модели [политика зарегистрирована](#registering-policies), то метод `can` автоматически вызовет соответствующую политику и вернет логический результат. Если для модели не зарегистрирована политика, то метод `can` попытается вызвать шлюз на основе замыкания, соответствующий переданному имени действия.

<a name="user-model-actions-that-dont-require-models"></a>
#### Авторизация действий, не требующих моделей, с помощью политик через модель User

Помните, что некоторые действия могут соответствовать методам политики, например `create`, которые не требуют экземпляра модели. В этих ситуациях вы можете передать имя класса методу `can`. Имя класса будет использоваться для определения того, какую политику использовать при авторизации действия:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Models\Post;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;

    class PostController extends Controller
    {
        /**
         * Сохранить пост.
         */
        public function store(Request $request): RedirectResponse
        {
            if ($request->user()->cannot('create', Post::class)) {
                abort(403);
            }

            // Сохранение поста...

            return redirect('/posts');
        }
    }

<a name="via-the-gate-facade"></a>
### Авторизация действий с помощью политик через через фасад `gate`

В дополнение к полезным методам, предоставляемым модели `App\Models\User`, вы всегда можете авторизовать действия с помощью метода `authorize` фасада `Gate`.

Подобно методу `can`, этот метод принимает имя действия, которое вы хотите авторизовать, и соответствующую модель. Если действие не авторизовано, то метод `authorize` выбросит исключение `Illuminate\Auth\Access\AuthorizationException`, которое обработчик исключений Laravel автоматически преобразует в `403` HTTP-ответ:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use App\Models\Post;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Gate;

    class PostController extends Controller
    {
        /**
         * Обновить переданный пост.
         *
         * @throws \Illuminate\Auth\Access\AuthorizationException
         */
        public function update(Request $request, Post $post): RedirectResponse
        {
            Gate::authorize('update', $post);

            // Текущий пользователь может обновить пост в блоге...

            return redirect('/posts');
        }
    }

<a name="controller-actions-that-dont-require-models"></a>
#### Авторизация действий, не требующих моделей, с помощью политик через помощников контроллера

Как обсуждалось ранее, некоторые методы политики, например `create`, не требуют экземпляра модели. В таких ситуациях вы должны передать имя класса методу `authorize`. Имя класса будет использоваться для определения того, какую политику использовать при авторизации действия:

    use App\Models\Post;
    use Illuminate\Http\RedirectResponse;
    use Illuminate\Http\Request;
    use Illuminate\Support\Facades\Gate;

    /**
     * Создайте новый пост в блоге.
     *
     * @throws \Illuminate\Auth\Access\AuthorizationException
     */
    public function create(Request $request): RedirectResponse
    {
        Gate::authorize('create', Post::class);

        // Текущий пользователь может создавать посты в блоге...

        return redirect('/posts');
    }

<a name="via-middleware"></a>
### Авторизация действий с помощью политик через посредника

Laravel содержит посредника, который может авторизовать действия до того, как входящий запрос достигнет ваших маршрутов или контроллеров. По умолчанию посреднику `Illuminate\Auth\Middleware\Authorize` может быть прикреплено к маршруту с помощью `can` [псевдоним промежуточного программного обеспечения](/docs/{{version}}/middleware#middleware-aliases), который автоматически регистрируется в Laravel. Давайте рассмотрим пример использования посредника `can` для авторизации того, что пользователь может обновлять пост:

    use App\Models\Post;

    Route::put('/post/{post}', function (Post $post) {
        // Текущий пользователь может обновить пост...
    })->middleware('can:update,post');

В этом примере мы передаем посреднику `can` два аргумента. Первый – это имя действия, которое мы хотим авторизовать, а второй – параметр маршрута, передаваемый методу политики. В этом случае поскольку мы используем [неявную привязку модели](/docs/{{version}}/routing#implicit-binding), то методу политики будет передана модель `App\Models\Post`. Если пользователь не авторизован для выполнения указанного действия, то посредник вернет ответ HTTP с кодом состояния `403`.

Для удобства вы также можете прикрепить посредник `can` к своему маршруту, используя метод `can`:

    use App\Models\Post;

    Route::put('/post/{post}', function (Post $post) {
        // Текущий пользователь может обновить сообщение...
    })->can('update', 'post');

<a name="middleware-actions-that-dont-require-models"></a>
#### Авторизация действий, не требующих моделей, с помощью политик через посредника

Опять же, некоторые методы политики, например `create`, не требуют экземпляра модели. В этих ситуациях вы можете передать имя класса посреднику. Имя класса будет использоваться для определения того, какую политику использовать при авторизации действия:

    Route::post('/post', function () {
        // Текущий пользователь может создавать посты...
    })->middleware('can:create,App\Models\Post');

Указание полного имени класса в определении посредника строки может стать обременительным. По этой причине вы можете присоединить посредник `can` к вашему маршруту, используя метод `can`:

    use App\Models\Post;

    Route::post('/post', function () {
        // Текущий пользователь может создавать сообщения...
    })->can('create', Post::class);

<a name="via-blade-templates"></a>
### Авторизация действий с помощью политик через шаблоны Blade

При написании шаблонов Blade бывает необходимо отобразить часть страницы только в том случае, если пользователь авторизован для выполнения конкретного действия. Например, вы можете показать форму обновления поста в блоге, только если пользователь действительно уполномочен обновить сообщение. В этой ситуации вы можете использовать директивы `@can` и `@cannot`:

```blade
@can('update', $post)
    <!-- Текущий пользователь может обновить пост... -->
@elsecan('create', App\Models\Post::class)
    <!-- Текущий пользователь может создавать новые посты... -->
@else
    <!-- ... -->
@endcan

@cannot('update', $post)
    <!-- Текущий пользователь не может обновить пост... -->
@elsecannot('create', App\Models\Post::class)
    <!-- Текущий пользователь не может создавать новые посты... -->
@endcannot
```

Эти директивы являются удобными ярлыками выражений `@if` и `@unless`. Приведенные выше директивы `@can` и `@cannot` эквивалентны следующим выражениям:

```blade
@if (Auth::user()->can('update', $post))
    <!-- Текущий пользователь может обновить пост... -->
@endif

@unless (Auth::user()->can('update', $post))
    <!-- Текущий пользователь не может обновить пост... -->
@endunless
```

Вы также можете определить, авторизован ли пользователь для выполнения любого из указанных в массиве действий. Для этого используйте директиву `@canany`:

```blade
@canany(['update', 'view', 'delete'], $post)
    <!-- Текущий пользователь может обновить, просмотреть или удалить пост ... -->
@elsecanany(['create'], \App\Models\Post::class)
    <!-- Текущий пользователь может создать пост... -->
@endcanany
```

<a name="blade-actions-that-dont-require-models"></a>
#### Авторизация действий, не требующих моделей, с помощью политик через шаблоны Blade

Как и большинство других методов авторизации, вы можете передать имя класса в директивы `@can` и `@cannot`, если для действия не требуется экземпляр модели:

```blade
@can('create', App\Models\Post::class)
    <!-- Текущий пользователь может создавать посты... -->
@endcan

@cannot('create', App\Models\Post::class)
    <!-- Текущий пользователь не может создавать посты... -->
@endcannot
```

<a name="supplying-additional-context"></a>
### Предоставление политикам дополнительного контекста

При авторизации действий с использованием политик вы можете передать массив в качестве второго аргумента различным функциям авторизации и помощникам. Первый элемент в массиве будет использоваться для определения того, какая политика должна быть вызвана, в то время как остальные элементы массива передаются как параметры методу политики и могут использоваться как дополнительный контекст при принятии решений об авторизации. Например, рассмотрим `PostPolicy` и следующее определение метода, содержащего дополнительный параметр `$category`:

    /**
     * Определить, может ли пользователь обновить пост.
     */
    public function update(User $user, Post $post, int $category): bool
    {
        return $user->id === $post->user_id &&
               $user->canUpdateCategory($category);
    }

При попытке определить, может ли аутентифицированный пользователь обновить указанный пост, мы можем вызвать этот метод политики следующим образом:

    /**
     * Обновить конкретный пост.
     *
     * @throws \Illuminate\Auth\Access\AuthorizationException
     */
    public function update(Request $request, Post $post): RedirectResponse
    {
        Gate::authorize('update', [$post, $request->category]);

        // Текущий пользователь может обновить пост в блоге...

        return redirect('/posts');
    }

<a name="authorization-and-inertia"></a>
## Авторизация и Inertia

Хотя авторизация всегда должна обрабатываться на сервере, часто бывает удобно предоставить вашему внешнему приложению данные авторизации, чтобы правильно отобразить пользовательский интерфейс вашего приложения. Laravel не определяет необходимое соглашение для предоставления информации об авторизации интерфейсу на базе Inertia.

Однако, если вы используете один из [стартовых наборов](/docs/{{version}}/starter-kit) Laravel на основе Inertia, ваше приложение уже содержит посредника `HandleInertiaRequests`. В рамках метода `share` этого посредника вы можете возвращать общие данные, которые будут предоставлены всем страницам Inertia в вашем приложении. Эти общие данные могут служить удобным местом для определения информации об авторизации пользователя:

```php
<?php

namespace App\Http\Middleware;

use App\Models\Post;
use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
    // ...

    /**
     * Определяем реквизиты, которые используются по умолчанию.
     *
     * @return array<string, mixed>
     */
    public function share(Request $request)
    {
        return [
            ...parent::share($request),
            'auth' => [
                'user' => $request->user(),
                'permissions' => [
                    'post' => [
                        'create' => $request->user()->can('create', Post::class),
                    ],
                ],
            ],
        ];
    }
}
```
