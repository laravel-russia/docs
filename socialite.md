---
git: 040ab151f8350470f8df5d00bfba05f56c155d12
---

# Пакет Laravel Socialite

<a name="introduction"></a>
## Введение

Помимо типичной аутентификации на основе форм, Laravel также предлагает простой и удобный способ аутентификации через провайдеров OAuth с помощью [Laravel Socialite](https://github.com/laravel/socialite). Socialite в настоящее время поддерживает аутентификацию через Facebook, X, LinkedIn, Google, GitHub, GitLab, Bitbucket и Slack.

> [!NOTE]
> Адаптеры для других платформ перечислены на веб-сайте [Socialite Providers](https://socialiteproviders.com/), управляемом сообществом.

<a name="installation"></a>
## Установка

Чтобы начать работу с Socialite, используйте менеджер пакетов Composer, чтобы добавить пакет в зависимости вашего проекта:

```shell
composer require laravel/socialite
```

<a name="upgrading-socialite"></a>
## Обновление пакета Socialite

При обновлении Socialite важно внимательно изучить [руководство по обновлению](https://github.com/laravel/socialite/blob/master/UPGRADE).

<a name="configuration"></a>
## Конфигурирование

Перед использованием Socialite вам нужно будет добавить учетные данные для провайдеров OAuth, которые использует ваше приложение. Обычно эти учетные данные можно получить, создав "приложение разработчика" в панели управления службы, с которой вы будете аутентифицироваться.

Эти учетные данные должны быть размещены в файле конфигурации вашего приложения `config/services.php` и должны использовать ключ `facebook`, `x`, `linkedin-openid`, `google`, `github`, `gitlab`, `bitbucket`, `slack` или `slack-openid`, в зависимости от провайдеров, которые требуются вашему приложению:

    'github' => [
        'client_id' => env('GITHUB_CLIENT_ID'),
        'client_secret' => env('GITHUB_CLIENT_SECRET'),
        'redirect' => 'http://example.com/callback-url',
    ],

> [!NOTE]
> Если параметр `redirect` содержит относительный путь, то он будет автоматически преобразован в абсолютный URL.

<a name="authentication"></a>
## Аутентификация

<a name="routing"></a>
### Маршрутизация

Для аутентификации пользователей с помощью провайдера OAuth вам понадобятся два маршрута: один для перенаправления пользователя к провайдеру OAuth, а другой для получения обратного вызова от провайдера после аутентификации. Пример ниже демонстрирует реализацию обоих маршрутов:

    use Laravel\Socialite\Facades\Socialite;

    Route::get('/auth/redirect', function () {
        return Socialite::driver('github')->redirect();
    });

    Route::get('/auth/callback', function () {
        $user = Socialite::driver('github')->user();

        // $user->token
    });

Метод `redirect` фасада `Socialite`, отвечает за перенаправление пользователя к провайдеру OAuth, в то время как метод `user` обрабатывает входящий запрос и получает информацию о пользователе от провайдера  после того, как запрос на аутентификацию будет подтверждён.

<a name="authentication-and-storage"></a>
### Аутентификация и хранение

После того как пользователь был получен от поставщика OAuth, вы можете определить, существует ли пользователь в базе данных вашего приложения и [аутентифицировать пользователя](/docs/{{version}}/authentication#authenticate-a-user-instance). Если пользователь не существует в базе данных вашего приложения, вы обычно создаете новую запись в своей базе данных:

    use App\Models\User;
    use Illuminate\Support\Facades\Auth;
    use Laravel\Socialite\Facades\Socialite;

    Route::get('/auth/callback', function () {
        $githubUser = Socialite::driver('github')->user();

        $user = User::updateOrCreate([
            'github_id' => $githubUser->id,
        ], [
            'name' => $githubUser->name,
            'email' => $githubUser->email,
            'github_token' => $githubUser->token,
            'github_refresh_token' => $githubUser->refreshToken,
        ]);

        Auth::login($user);

        return redirect('/dashboard');
    });

> [!NOTE]
> Для получения дополнительной информации о том, какая информация о пользователях доступна от конкретных поставщиков OAuth, обратитесь к документации по [получению сведений о пользователе](#retrieving-user-details).

<a name="access-scopes"></a>
### Права доступа

Перед перенаправлением пользователя вы можете использовать метод `scopes`, чтобы указать "scopes" (права/области) которые должны быть включены в запрос аутентификации.  Этот метод объединит все ранее указанные права с теми, которые указали вы:

    use Laravel\Socialite\Facades\Socialite;

    return Socialite::driver('github')
        ->scopes(['read:user', 'public_repo'])
        ->redirect();

Вы можете перезаписать все существующие права в запросе аутентификации, используя метод `setScopes`:

    return Socialite::driver('github')
        ->setScopes(['read:user', 'public_repo'])
        ->redirect();

<a name="slack-bot-scopes"></a>
### Права Slack Bot

API Slack предоставляет [разные типы токенов доступа](https://api.slack.com/authentication/token-types), каждый с собственным набором [прав](https://api.slack.com/scopes). Socialite совместим с обоими следующими типами токенов доступа Slack:

<!-- <div class="content-list" markdown="1"> -->

- Bot (prefixed with `xoxb-`)
- User (prefixed with `xoxp-`)

<!-- </div> -->

По умолчанию драйвер `slack` создаст токен `user`, и вызов метода `user` этого драйвера вернет данные пользователя.

Токены ботов в основном полезны, если ваше приложение будет отправлять уведомления во внешние рабочие пространства Slack, принадлежащие пользователям вашего приложения. Чтобы сгенерировать токен бота, вызовите метод `asBotUser` перед перенаправлением пользователя в Slack для аутентификации:

    return Socialite::driver('slack')
        ->asBotUser()
        ->setScopes(['chat:write', 'chat:write.public', 'chat:write.customize'])
        ->redirect();

Кроме того, вы должны вызвать метод `asBotUser` перед вызовом метода `user`, когда Slack перенаправляет пользователя обратно на ваше приложение после аутентификации:

    $user = Socialite::driver('slack')->asBotUser()->user();

При генерации токена бота метод `user` по-прежнему будет возвращать экземпляр `Laravel\Socialite\Two\User`, однако только свойство `token` будет заполнено. Этот токен можно сохранить, чтобы [отправлять уведомления в рабочие пространства Slack аутентифицированного пользователя](/docs/{{version}}/notifications#notifying-external-slack-workspaces).

<a name="optional-parameters"></a>
### Необязательные параметры

Некоторые провайдеры OAuth поддерживают другие необязательные параметры в запросе перенаправления. Чтобы включить в запрос любые необязательные параметры, вызовите метод `with` с ассоциативным массивом:

    use Laravel\Socialite\Facades\Socialite;

    return Socialite::driver('google')
        ->with(['hd' => 'example.com'])
        ->redirect();

> [!WARNING]
> При использовании метода `with` будьте осторожны, чтобы не передавать какие-либо зарезервированные ключевые слова, такие как `state` или `response_type`.

<a name="retrieving-user-details"></a>
## Получение сведений о пользователе

После того как пользователь будет перенаправлен обратно на ваш маршрут `callback` аутентификации вашего приложения, вы можете получить данные пользователя, используя метод `user` пакета Socialite. Объект пользователя, возвращаемый методом `user`, содержит множество свойств и методов, которые вы можете использовать для сохранения информации о пользователе в вашей собственной базе данных.

Различные свойства и методы этого объекта могут быть доступны в зависимости от версии провайдера OAuth, с которым вы выполняете аутентификацию, OAuth 1.0 или OAuth 2.0:

    use Laravel\Socialite\Facades\Socialite;

    Route::get('/auth/callback', function () {
        $user = Socialite::driver('github')->user();

        // Провайдер OAuth 2.0 ...
        $token = $user->token;
        $refreshToken = $user->refreshToken;
        $expiresIn = $user->expiresIn;

        // Провайдер OAuth 1.0 ...
        $token = $user->token;
        $tokenSecret = $user->tokenSecret;

        // Все провайдеры ...
        $user->getId();
        $user->getNickname();
        $user->getName();
        $user->getEmail();
        $user->getAvatar();
    });

<a name="retrieving-user-details-from-a-token-oauth2"></a>
#### Получение сведений о пользователе из токена

Если у вас уже есть действительный токен доступа пользователя, то вы можете получить его данные с помощью метода `userFromToken` пакета Socialite:

    use Laravel\Socialite\Facades\Socialite;

    $user = Socialite::driver('github')->userFromToken($token);

Если вы используете ограниченный вход в Facebook через приложение iOS, Facebook вернет токен OIDC вместо токена доступа. Как и токен доступа, токен OIDC может быть предоставлен методу `userFromToken` для получения сведений о пользователе.

<a name="stateless-authentication"></a>
#### Аутентификация без сохранения состояния

Метод `stateless` используется для отключения проверки состояния сессии. Это полезно при добавлении социальной аутентификации в API без сохранения состояния, не использующему сеансы на основе файлов cookie::

    use Laravel\Socialite\Facades\Socialite;

    return Socialite::driver('google')->stateless()->user();
