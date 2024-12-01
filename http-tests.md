---
git: b874bc07a34f0a9c960f3e1b7ced2370724abcf9
---

# Тестирование · Тесты HTTP


<a name="introduction"></a>
## Введение

Laravel предлагает гибкий API в составе вашего приложения для выполнения HTTP-запросов и получения информации об ответах. Например, взгляните на следующий функциональный тест:

```php tab=Pest
<?php

test('the application returns a successful response', function () {
    $response = $this->get('/');

    $response->assertStatus(200);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * Отвлеченный пример функционального теста.
     */
    public function test_the_application_returns_a_successful_response(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

Метод `get` отправляет в приложение запрос `GET`, а метод `assertStatus` утверждает, что возвращаемый ответ должен иметь указанный код состояния HTTP. Помимо этого простого утверждения, Laravel также содержит множество утверждений для получения информации о заголовках ответов, их содержимого, структуры JSON и др.

<a name="making-requests"></a>
## Выполнение запросов

Чтобы сделать запрос к вашему приложению, вы можете вызвать в своем тесте методы `get`, `post`, `put`, `patch`, или `delete`. Эти методы фактически не отправляют вашему приложению «настоящий» HTTP-запрос. Вместо этого внутри моделируется полный сетевой запрос.

Вместо того чтобы возвращать экземпляр `Illuminate\Http\Response`, методы тестового запроса возвращают экземпляр `Illuminate\Testing\TestResponse`, который содержит [множество полезных утверждений](#available-assertions), позволяющие вам инспектировать ответы вашего приложения:

```php tab=Pest
<?php

test('basic request', function () {
    $response = $this->get('/');

    $response->assertStatus(200);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_a_basic_request(): void
    {
        $response = $this->get('/');

        $response->assertStatus(200);
    }
}
```

Как правило, каждый из ваших тестов должен выполнять только один запрос к вашему приложению. Неожиданное поведение может возникнуть, если в рамках одного метода теста выполняется несколько запросов.

> [!NOTE]
> Для удобства посредник CSRF автоматически отключается при запуске тестов.

<a name="customizing-request-headers"></a>
### Настройка заголовков запросов

Вы можете использовать метод `withHeaders` для настройки заголовков запроса перед его отправкой в приложение. Этот метод позволяет вам добавлять в запрос любые пользовательские заголовки:

```php tab=Pest
<?php

test('interacting with headers', function () {
    $response = $this->withHeaders([
        'X-Header' => 'Value',
    ])->post('/user', ['name' => 'Sally']);

    $response->assertStatus(201);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_interacting_with_headers(): void
    {
        $response = $this->withHeaders([
            'X-Header' => 'Value',
        ])->post('/user', ['name' => 'Sally']);

        $response->assertStatus(201);
    }
}
```

<a name="cookies"></a>
### Cookies

Вы можете использовать методы `withCookie` или `withCookies` для установки значений файлов Cookies перед отправкой запроса. Метод `withCookie` принимает имя и значение Cookie в качестве двух аргументов, а метод `withCookies` принимает массив пар имя / значение:

```php tab=Pest
<?php

test('interacting with cookies', function () {
    $response = $this->withCookie('color', 'blue')->get('/');

    $response = $this->withCookies([
        'color' => 'blue',
        'name' => 'Taylor',
    ])->get('/');

    //
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_interacting_with_cookies(): void
    {
        $response = $this->withCookie('color', 'blue')->get('/');

        $response = $this->withCookies([
            'color' => 'blue',
            'name' => 'Taylor',
        ])->get('/');

        //
    }
}
```

<a name="session-and-authentication"></a>
### Сессия / Аутентификация

Laravel предлагает несколько методов-хелперов для взаимодействия с сессией во время HTTP-тестирования. Во-первых, вы можете установить данные сессии, передав массив, используя метод `withSession`. Это полезно для загрузки сессии данными перед отправкой запроса вашему приложению:

```php tab=Pest
<?php

test('interacting with the session', function () {
    $response = $this->withSession(['banned' => false])->get('/');

    //
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_interacting_with_the_session(): void
    {
        $response = $this->withSession(['banned' => false])->get('/');

        //
    }
}
```

Сессия Laravel обычно используется для сохранения состояния текущего аутентифицированного пользователя. Вспомогательный метод `actingAs` – это простой способ аутентифицировать конкретного пользователя как текущего. Например, мы можем использовать [фабрику модели](/docs/{{version}}/eloquent-factories) для генерации и аутентификации пользователя:

```php tab=Pest
<?php

use App\Models\User;

test('an action that requires authentication', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)
                     ->withSession(['banned' => false])
                     ->get('/');

    //
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Models\User;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_an_action_that_requires_authentication(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
                         ->withSession(['banned' => false])
                         ->get('/');

        //
    }
}
```

Вы также можете указать, какой гейт должен использоваться для аутентификации конкретного пользователя, передав имя гейта в качестве второго аргумента методу `actingAs`.  Гейт, предоставленный методу actingAs, также станет гейтом по умолчанию на протяжении всего теста::

    $this->actingAs($user, 'web')

<a name="debugging-responses"></a>
### Отладка ответов

После выполнения тестового запроса к вашему приложению методы `dump`, `dumpHeaders`, и `dumpSession` могут быть использованы для проверки и отладки содержимого ответа:

```php tab=Pest
<?php

test('basic test', function () {
    $response = $this->get('/');

    $response->dumpHeaders();

    $response->dumpSession();

    $response->dump();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_basic_test(): void
    {
        $response = $this->get('/');

        $response->dumpHeaders();

        $response->dumpSession();

        $response->dump();
    }
}
```

В качестве альтернативы вы можете использовать методы `dd`, `ddHeaders` и `ddSession`, чтобы выгрузить информацию об ответе и затем остановить выполнение:

```php tab=Pest
<?php

test('basic test', function () {
    $response = $this->get('/');

    $response->ddHeaders();

    $response->ddSession();

    $response->dd();
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_basic_test(): void
    {
        $response = $this->get('/');

        $response->ddHeaders();

        $response->ddSession();

        $response->dd();
    }
}
```

<a name="exception-handling"></a>
### Обработка исключений

Иногда вам может понадобиться проверить, выдает ли ваше приложение определенное исключение. Для этого вы можете «подделать» обработчик исключений через фасад `Exceptions`. После того как обработчик исключений был подделан, вы можете использовать методы `assertReported` и `assertNotReported` для создания утверждений против исключений, которые были созданы во время запроса:

```php tab=Pest
<?php

use App\Exceptions\InvalidOrderException;
use Illuminate\Support\Facades\Exceptions;

test('exception is thrown', function () {
    Exceptions::fake();

    $response = $this->get('/order/1');

    // Assert an exception was thrown...
    Exceptions::assertReported(InvalidOrderException::class);

    // Assert against the exception...
    Exceptions::assertReported(function (InvalidOrderException $e) {
        return $e->getMessage() === 'The order was invalid.';
    });
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use App\Exceptions\InvalidOrderException;
use Illuminate\Support\Facades\Exceptions;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic test example.
     */
    public function test_exception_is_thrown(): void
    {
        Exceptions::fake();

        $response = $this->get('/');

        // Assert an exception was thrown...
        Exceptions::assertReported(InvalidOrderException::class);

        // Assert against the exception...
        Exceptions::assertReported(function (InvalidOrderException $e) {
            return $e->getMessage() === 'The order was invalid.';
        });
    }
}
```

Методы `assertNotReported` и `assertNothingReported` могут использоваться для подтверждения того, что данное исключение не было создано во время запроса или что никаких исключений не было создано:

```php
Exceptions::assertNotReported(InvalidOrderException::class);

Exceptions::assertNothingReported();
```

Вы можете полностью отключить обработку исключений для данного запроса, вызвав метод `withoutExceptionHandling` перед отправкой запроса:

    $response = $this->withoutExceptionHandling()->get('/');

Кроме того, если вы хотите убедиться, что ваше приложение не использует функции, считающиеся устаревшими языком PHP или библиотеками, которые использует ваше приложение, вы можете вызвать метод `withoutDeprecationHandling` перед тем, как сделать свой запрос. Когда обработка устаревания отключена, предупреждения об устаревании будут преобразованы в исключения, что приведет к сбою вашего теста:

    $response = $this->withoutDeprecationHandling()->get('/');

Метод assertThrows можно использовать для проверки того, что код внутри заданного замыкания генерирует исключение `указанного` типа:

```php
$this->assertThrows(
    fn () => (new ProcessOrder)->execute(),
    OrderInvalid::class
);
```

Если вы хотите проверить и сделать утверждения против выброшенного исключения, вы можете предоставить замыкание в качестве второго аргумента метода `assertThrows`:

```php
$this->assertThrows(
    fn () => (new ProcessOrder)->execute(),
    fn (OrderInvalid $e) => $e->orderId() === 123;
);
```

<a name="testing-json-apis"></a>
## Тестирование JSON API

Laravel также содержит несколько хелперов для тестирования API-интерфейсов JSON и их ответов. Например, методы `json`, `getJson`, `postJson`, `putJson`, `patchJson`, `deleteJson`, и `optionsJson` могут использоваться для отправки запросов JSON с различными HTTP-командами. Вы также можете передавать данные и заголовки этим методам. Для начала давайте напишем тест, чтобы сделать запрос `POST` к `/api/user` и убедиться, что в JSON были возвращены ожидаемые данные:

```php tab=Pest
<?php

test('making an api request', function () {
    $response = $this->postJson('/api/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertJson([
            'created' => true,
         ]);
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_making_an_api_request(): void
    {
        $response = $this->postJson('/api/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertJson([
                'created' => true,
            ]);
    }
}
```

Кроме того, к данным ответа JSON можно получить доступ как к переменным массива в ответе, что позволяет удобно проверять отдельные значения, возвращаемые в JSON-ответе:

```php tab=Pest
expect($response['created'])->toBeTrue();
```

```php tab=PHPUnit
$this->assertTrue($response['created']);
```

> [!NOTE]
> Метод `assertJson` преобразует ответ в массив для проверки того, что переданный массив существует в ответе JSON, возвращаемом приложением. Итак, если в ответе JSON есть другие свойства, этот тест все равно будет проходить, пока присутствует переданный фрагмент.

<a name="verifying-exact-match"></a>
#### Утверждение точных совпадений JSON

Как упоминалось ранее, метод `assertJson` используется для подтверждения наличия фрагмента JSON в ответе JSON. Если вы хотите убедиться, что данный массив **в точности соответствует** JSON, возвращаемому вашим приложением, вы должны использовать метод `assertExactJson`:

```php tab=Pest
<?php

test('asserting an exact json match', function () {
    $response = $this->postJson('/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertExactJson([
            'created' => true,
        ]);
});

```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_asserting_an_exact_json_match(): void
    {
        $response = $this->postJson('/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertExactJson([
                'created' => true,
            ]);
    }
}
```

<a name="verifying-json-paths"></a>
#### Утверждения в JSON-путях

Если вы хотите убедиться, что ответ JSON содержит данные по указанному пути, вам следует использовать метод `assertJsonPath`:

```php tab=Pest
<?php

test('asserting a json path value', function () {
    $response = $this->postJson('/user', ['name' => 'Sally']);

    $response
        ->assertStatus(201)
        ->assertJsonPath('team.owner.name', 'Darian');
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    /**
     * A basic functional test example.
     */
    public function test_asserting_a_json_paths_value(): void
    {
        $response = $this->postJson('/user', ['name' => 'Sally']);

        $response
            ->assertStatus(201)
            ->assertJsonPath('team.owner.name', 'Darian');
    }
}
```

Метод `assertJsonPath` также принимает замыкание, которое может быть использовано для динамического определения, должно ли утверждение выполниться:

    $response->assertJsonPath('team.owner.name', fn (string $name) => strlen($name) >= 3);

<a name="fluent-json-testing"></a>
### Последовательное тестирование JSON

Laravel предлагает способ последовательного тестирования ответов JSON вашего приложения. Для начала передайте замыкание методу `assertJson`. Это замыкание будет вызываться с экземпляром класса `Illuminate\Testing\Fluent\AssertableJson`, который можно использовать для создания утверждений в отношении JSON, возвращенного вашим приложением. Метод `where` может использоваться для утверждения определенного атрибута JSON, в то время как метод `missing` может использоваться для утверждения отсутствия конкретного атрибута в JSON:

```php tab=Pest
use Illuminate\Testing\Fluent\AssertableJson;

test('fluent json', function () {
    $response = $this->getJson('/users/1');

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->where('id', 1)
                 ->where('name', 'Victoria Faith')
                 ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                 ->whereNot('status', 'pending')
                 ->missing('password')
                 ->etc()
        );
});
```

```php tab=PHPUnit
use Illuminate\Testing\Fluent\AssertableJson;

/**
 * A basic functional test example.
 */
public function test_fluent_json(): void
{
    $response = $this->getJson('/users/1');

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->where('id', 1)
                 ->where('name', 'Victoria Faith')
                 ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                 ->whereNot('status', 'pending')
                 ->missing('password')
                 ->etc()
        );
}
```

#### Понимание метода `etc`

В приведенном выше примере вы могли заметить, что мы вызвали метод `etc` в конце нашей цепочки утверждений. Этот метод сообщает Laravel, что в объекте JSON могут присутствовать другие атрибуты. Если метод `etc` не используется, то тест завершится неудачно, если в объекте JSON существуют другие атрибуты, для которых вы не сделали утверждений.

Цель такого поведения – защитить вас от непреднамеренного раскрытия конфиденциальной информации в ваших ответах JSON, заставив вас либо явно сделать утверждение относительно атрибута, либо явно разрешить дополнительные атрибуты с помощью метода `etc`.

Однако вы должны знать, что отсутствие метода `etc` в вашей цепочке утверждений не гарантирует, что дополнительные атрибуты не будут добавлены в массивы, вложенные в ваш объект JSON. Метод `etc` обеспечивает только отсутствие дополнительных атрибутов на уровне вложенности, на котором вызывается метод `etc`.

<a name="asserting-json-attribute-presence-and-absence"></a>
#### Утверждение наличия / отсутствия атрибута

Чтобы утверждать, что атрибут присутствует или отсутствует, вы можете использовать методы `has` и `missing`:

    $response->assertJson(fn (AssertableJson $json) =>
        $json->has('data')
             ->missing('message')
    );

Кроме того, методы `hasAll` и `missingAll` позволяют одновременно утверждать наличие или отсутствие нескольких атрибутов:

    $response->assertJson(fn (AssertableJson $json) =>
        $json->hasAll(['status', 'data'])
             ->missingAll(['message', 'code'])
    );

Вы можете использовать метод `hasAny`, чтобы определить, присутствует ли хотя бы один из заданного списка атрибутов:

    $response->assertJson(fn (AssertableJson $json) =>
        $json->has('status')
             ->hasAny('data', 'message', 'code')
    );

<a name="asserting-against-json-collections"></a>
#### Утверждения относительно коллекций JSON

Часто ваш маршрут возвращает ответ JSON, содержащий несколько элементов, например нескольких пользователей:

    Route::get('/users', function () {
        return User::all();
    });

В этих ситуациях можно использовать метод `has` последовательного тестирования JSON, чтобы сделать утверждения относительно пользователей, содержащихся в ответе. Например, предположим, что ответ JSON содержит трех пользователей. Затем мы сделаем некоторые утверждения относительно первого пользователя в коллекции, используя метод `first`. Метод `first` принимает замыкание, получающее другой экземпляр `AssertableJson`, который можно использовать для создания утверждений относительно первого объекта коллекции JSON:

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has(3)
                 ->first(fn (AssertableJson $json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                         ->missing('password')
                         ->etc()
                 )
        );

<a name="scoping-json-collection-assertions"></a>
#### Уровень вложенности утверждения относительно коллекций JSON

Иногда маршрутами вашего приложения могут быть возвращены коллекции JSON, которым назначены именованные ключи:

    Route::get('/users', function () {
        return [
            'meta' => [...],
            'users' => User::all(),
        ];
    })

При тестировании этих маршрутов вы можете использовать метод `has` для утверждения относительно количества элементов в коллекции. Кроме того, вы можете использовать метод `has` для определения цепочки утверждений:

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has('meta')
                 ->has('users', 3)
                 ->has('users.0', fn (AssertableJson $json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                         ->missing('password')
                         ->etc()
                 )
        );

Однако вместо того, чтобы делать два отдельных вызова метода `has` для утверждения в отношении коллекции `users`, вы можете сделать один вызов, обеспеченный замыканием в качестве третьего параметра. При этом автоматически вызывается замыкание, область действия которого будет ограниченно уровнем вложенности первого элемента коллекции:

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has('meta')
                 ->has('users', 3, fn (AssertableJson $json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->where('email', fn (string $email) => str($email)->is('victoria@gmail.com'))
                         ->missing('password')
                         ->etc()
                 )
        );

<a name="asserting-json-types"></a>
#### Утверждения относительно типов JSON

При необходимости можно утверждать, что свойства в ответе JSON имеют определенный тип. Класс `Illuminate\Testing\Fluent\AssertableJson` содержит методы `whereType` и `whereAllType`, обеспечивающие простоту таких утверждений:

    $response->assertJson(fn (AssertableJson $json) =>
        $json->whereType('id', 'integer')
             ->whereAllType([
                'users.0.name' => 'string',
                'meta' => 'array'
            ])
    );

Можно указать несколько типов в качестве второго параметра метода `whereType`, разделив их символом `|`, или передав массив необходимых типов. Утверждение будет успешно, если значение ответа будет иметь какой-либо из перечисленных типов:

    $response->assertJson(fn (AssertableJson $json) =>
        $json->whereType('name', 'string|null')
             ->whereType('id', ['string', 'integer'])
    );

Методы `whereType` и `whereAllType` применимы к следующим типам: `string`, `integer`, `double`, `boolean`, `array`, и `null`.

<a name="testing-file-uploads"></a>
## Тестирование загрузки файлов

Класс `Illuminate\Http\UploadedFile` содержит метод `fake`, который можно использовать для создания фиктивных файлов или изображений для тестирования. Это, в сочетании с методом `fake` фасада `Storage`, значительно упрощает тестирование загрузки файлов. Например, вы можете объединить эти две функции, чтобы легко протестировать форму загрузки аватара:

```php tab=Pest
<?php

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

test('avatars can be uploaded', function () {
    Storage::fake('avatars');

    $file = UploadedFile::fake()->image('avatar.jpg');

    $response = $this->post('/avatar', [
        'avatar' => $file,
    ]);

    Storage::disk('avatars')->assertExists($file->hashName());
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_avatars_can_be_uploaded(): void
    {
        Storage::fake('avatars');

        $file = UploadedFile::fake()->image('avatar.jpg');

        $response = $this->post('/avatar', [
            'avatar' => $file,
        ]);

        Storage::disk('avatars')->assertExists($file->hashName());
    }
}
```

Если вы хотите подтвердить, что переданный файл не существует, вы можете использовать метод `assertMissing` фасада `Storage`:

    Storage::fake('avatars');

    // ...

    Storage::disk('avatars')->assertMissing('missing.jpg');

<a name="fake-file-customization"></a>
#### Настройка фиктивного файла

При создании файлов с использованием метода `fake`, предоставляемого классом `UploadedFile`, вы можете указать ширину, высоту и размер изображения (в килобайтах), чтобы лучше протестировать правила валидации вашего приложения:

    UploadedFile::fake()->image('avatar.jpg', $width, $height)->size(100);

Помимо создания изображений, вы можете создавать файлы любого другого типа, используя метод `create`:

    UploadedFile::fake()->create('document.pdf', $sizeInKilobytes);

При необходимости вы можете передать аргумент `$mimeType` методу, чтобы явно определить MIME-тип, который должен возвращать файл:

    UploadedFile::fake()->create(
        'document.pdf', $sizeInKilobytes, 'application/pdf'
    );

<a name="testing-views"></a>
## Тестирование шаблонной системы

Laravel также позволяет отображать шаблоны без имитации HTTP-запроса к приложению. Для этого вы можете вызвать в своем тесте метод `view`. Метод `view` принимает имя шаблона и необязательный массив данных. Метод возвращает экземпляр `Illuminate\Testing\TestView`, который предлагает несколько методов для удобных утверждений о содержимом шаблона:

```php tab=Pest
<?php

test('a welcome view can be rendered', function () {
    $view = $this->view('welcome', ['name' => 'Taylor']);

    $view->assertSee('Taylor');
});
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Tests\TestCase;

class ExampleTest extends TestCase
{
    public function test_a_welcome_view_can_be_rendered(): void
    {
        $view = $this->view('welcome', ['name' => 'Taylor']);

        $view->assertSee('Taylor');
    }
}
```

Класс `TestView` содержит следующие методы утверждения: `assertSee`, `assertSeeInOrder`, `assertSeeText`, `assertSeeTextInOrder`, `assertDontSee` и `assertDontSeeText`.

При необходимости вы можете получить необработанное отрисованное содержимое шаблона, преобразовав экземпляр `TestView` в строку:

    $contents = (string) $this->view('welcome');

<a name="sharing-errors"></a>
#### Передача ошибок валидации в шаблоны

Некоторые шаблоны могут зависеть от ошибок, хранящихся в [глобальной коллекции ошибок Laravel](/docs/{{version}}/validation#quick-displaying-the-validation-errors). Чтобы добавить в эту коллекцию сообщения об ошибках, вы можете использовать метод `withViewErrors`:

    $view = $this->withViewErrors([
        'name' => ['Please provide a valid name.']
    ])->view('form');

    $view->assertSee('Please provide a valid name.');

<a name="rendering-blade-and-components"></a>
### Отрисовка Blade и компоненты

Если необходимо, вы можете использовать метод `blade` для анализа и отрисовки необработанной строки [Blade](/docs/{{version}}/blade). Подобно методу `view`, метод `blade` возвращает экземпляр `Illuminate\Testing\TestView`:

    $view = $this->blade(
        '<x-component :name="$name" />',
        ['name' => 'Taylor']
    );

    $view->assertSee('Taylor');

Вы можете использовать метод `component` для анализа и отрисовки [компонента Blade](/docs/{{version}}/blade#components). Метод `component` возвращает экземпляр `Illuminate\Testing\TestComponent`:

    $view = $this->component(Profile::class, ['name' => 'Taylor']);

    $view->assertSee('Taylor');

<a name="available-assertions"></a>
## Доступные утверждения

<a name="response-assertions"></a>
### Утверждения ответов

Класс `Illuminate\Testing\TestResponse` содержит множество своих методов утверждения, которые вы можете использовать при тестировании вашего приложения. К этим утверждениям можно получить доступ в ответе, возвращаемом тестовыми методами `json`, `get`, `post`, `put`, и `delete`:

<div class="docs-column-list-2" markdown="1">

- [assertAccepted](#assert-accepted)
- [assertBadRequest](#assert-bad-request)
- [assertConflict](#assert-conflict)
- [assertCookie](#assert-cookie)
- [assertCookieExpired](#assert-cookie-expired)
- [assertCookieNotExpired](#assert-cookie-not-expired)
- [assertCookieMissing](#assert-cookie-missing)
- [assertCreated](#assert-created)
- [assertDontSee](#assert-dont-see)
- [assertDontSeeText](#assert-dont-see-text)
- [assertDownload](#assert-download)
- [assertExactJson](#assert-exact-json)
- [assertExactJsonStructure](#assert-exact-json-structure)
- [assertForbidden](#assert-forbidden)
- [assertFound](#assert-found)
- [assertGone](#assert-gone)
- [assertHeader](#assert-header)
- [assertHeaderMissing](#assert-header-missing)
- [assertInternalServerError](#assert-internal-server-error)
- [assertJson](#assert-json)
- [assertJsonCount](#assert-json-count)
- [assertJsonFragment](#assert-json-fragment)
- [assertJsonIsArray](#assert-json-is-array)
- [assertJsonIsObject](#assert-json-is-object)
- [assertJsonMissing](#assert-json-missing)
- [assertJsonMissingExact](#assert-json-missing-exact)
- [assertJsonMissingValidationErrors](#assert-json-missing-validation-errors)
- [assertJsonPath](#assert-json-path)
- [assertJsonMissingPath](#assert-json-missing-path)
- [assertJsonStructure](#assert-json-structure)
- [assertJsonValidationErrors](#assert-json-validation-errors)
- [assertJsonValidationErrorFor](#assert-json-validation-error-for)
- [assertLocation](#assert-location)
- [assertMethodNotAllowed](#assert-method-not-allowed)
- [assertMovedPermanently](#assert-moved-permanently)
- [assertContent](#assert-content)
- [assertNoContent](#assert-no-content)
- [assertStreamedContent](#assert-streamed-content)
- [assertNotFound](#assert-not-found)
- [assertOk](#assert-ok)
- [assertPaymentRequired](#assert-payment-required)
- [assertPlainCookie](#assert-plain-cookie)
- [assertRedirect](#assert-redirect)
- [assertRedirectContains](#assert-redirect-contains)
- [assertRedirectToRoute](#assert-redirect-to-route)
- [assertRedirectToSignedRoute](#assert-redirect-to-signed-route)
- [assertRequestTimeout](#assert-request-timeout)
- [assertSee](#assert-see)
- [assertSeeInOrder](#assert-see-in-order)
- [assertSeeText](#assert-see-text)
- [assertSeeTextInOrder](#assert-see-text-in-order)
- [assertServerError](#assert-server-error)
- [assertServiceUnavailable](#assert-server-unavailable)
- [assertSessionHas](#assert-session-has)
- [assertSessionHasInput](#assert-session-has-input)
- [assertSessionHasAll](#assert-session-has-all)
- [assertSessionHasErrors](#assert-session-has-errors)
- [assertSessionHasErrorsIn](#assert-session-has-errors-in)
- [assertSessionHasNoErrors](#assert-session-has-no-errors)
- [assertSessionDoesntHaveErrors](#assert-session-doesnt-have-errors)
- [assertSessionMissing](#assert-session-missing)
- [assertStatus](#assert-status)
- [assertSuccessful](#assert-successful)
- [assertTooManyRequests](#assert-too-many-requests)
- [assertUnauthorized](#assert-unauthorized)
- [assertUnprocessable](#assert-unprocessable)
- [assertUnsupportedMediaType](#assert-unsupported-media-type)
- [assertValid](#assert-valid)
- [assertInvalid](#assert-invalid)
- [assertViewHas](#assert-view-has)
- [assertViewHasAll](#assert-view-has-all)
- [assertViewIs](#assert-view-is)
- [assertViewMissing](#assert-view-missing)

</div>

<a name="assert-bad-request"></a>
#### assertBadRequest

Утверждает, что ответ имеет код `400` состояния HTTP – `bad request`:

    $response->assertBadRequest();

<a name="assert-accepted"></a>
#### assertAccepted

Утверждает, что ответ имеет код `202` состояния HTTP – `accepted`:

    $response->assertAccepted();

<a name="assert-conflict"></a>
#### assertConflict

Утверждает, что ответ имеет код `409` состояния HTTP – `conflict`:

    $response->assertConflict();

<a name="assert-cookie"></a>
#### assertCookie

Утверждает, что ответ содержит переданный cookie:

    $response->assertCookie($cookieName, $value = null);

<a name="assert-cookie-expired"></a>
#### assertCookieExpired

Утверждает, что в ответе содержится переданный cookie и срок его действия истек:

    $response->assertCookieExpired($cookieName);

<a name="assert-cookie-not-expired"></a>
#### assertCookieNotExpired

Утверждает, что в ответе содержится переданный cookie и срок его действия не истек:

    $response->assertCookieNotExpired($cookieName);

<a name="assert-cookie-missing"></a>
#### assertCookieMissing

Утверждает, что ответ не содержит переданный cookie:

    $response->assertCookieMissing($cookieName);

<a name="assert-created"></a>
#### assertCreated

Утверждает, что ответ имеет код `201` состояния HTTP:

    $response->assertCreated();

<a name="assert-dont-see"></a>
#### assertDontSee

Утверждает, что переданная строка не содержится в ответе, возвращаемом приложением. Это утверждение автоматически экранирует переданную строку, если вы не передадите второй аргумент как `false`:

    $response->assertDontSee($value, $escaped = true);

<a name="assert-dont-see-text"></a>
#### assertDontSeeText

Утверждает, что переданная строка не содержится в тексте ответа. Это утверждение автоматически экранирует переданную строку, если вы не передадите второй аргумент как `false`. Этот метод передаст содержимое ответа PHP-функции `strip_tags` перед тем, как выполнить утверждение:

    $response->assertDontSeeText($value, $escaped = true);

<a name="assert-download"></a>
#### assertDownload

Утверждение, что ответ является отдачей файла. Обычно это означает, что вызванный маршрут, который вернул ответ, вернул ответ `Response::download`, `BinaryFileResponse` или `Storage::download`:

    $response->assertDownload();

При желании вы можете сделать утверждение, что загружаемому файлу было присвоено данное имя файла:

    $response->assertDownload('image.jpg');

<a name="assert-exact-json"></a>
#### assertExactJson

Утверждает, что ответ содержит точное совпадение указанных данных JSON:

    $response->assertExactJson(array $data);

<a name="assert-exact-json-structure"></a>
#### assertExactJsonStructure

Убедитесь, что ответ содержит точное соответствие заданной структуре JSON:

    $response->assertExactJsonStructure(array $data);

Этот метод является более строгим вариантом [assertJsonStructure](#assert-json-structure). В отличие от `assertJsonStructure`, этот метод завершится ошибкой, если ответ содержит какие-либо ключи, которые явно не включены в ожидаемую структуру JSON.

<a name="assert-forbidden"></a>
#### assertForbidden

Утверждает, что ответ имеет код `403` состояния HTTP – `forbidden`:

    $response->assertForbidden();

<a name="assert-found"></a>
#### assertFound

Утверждает, что ответ имеет код `302` состояния HTTP – `found`:

    $response->assertFound();

<a name="assert-gone"></a>
#### assertGone

Утверждает, что ответ имеет код `420` состояния HTTP – `gone`:

    $response->assertGone();

<a name="assert-header"></a>
#### assertHeader

Утверждает, что переданный заголовок и значение присутствуют в ответе:

    $response->assertHeader($headerName, $value = null);

<a name="assert-header-missing"></a>
#### assertHeaderMissing

Утверждает, что переданный заголовок отсутствует в ответе:

    $response->assertHeaderMissing($headerName);

<a name="assert-internal-server-error"></a>
#### assertInternalServerError

Утверждает, что ответ имеет код `500` состояния HTTP – `Internal Server Error`:

    $response->assertInternalServerError();

<a name="assert-json"></a>
#### assertJson

Утверждает, что ответ содержит указанные данные JSON:

    $response->assertJson(array $data, $strict = false);

Метод `assertJson` преобразует ответ в массив для проверки того, что переданный массив существует в ответе JSON, возвращаемом приложением. Итак, если в ответе JSON есть другие свойства, этот тест все равно будет проходить, пока присутствует переданный фрагмент.

<a name="assert-json-count"></a>
#### assertJsonCount

Утверждает, что ответ JSON имеет массив с ожидаемым количеством элементов указанного ключа:

    $response->assertJsonCount($count, $key = null);

<a name="assert-json-fragment"></a>
#### assertJsonFragment

Утверждает, что ответ содержит указанные данные JSON в любом месте ответа:

    Route::get('/users', function () {
        return [
            'users' => [
                [
                    'name' => 'Taylor Otwell',
                ],
            ],
        ];
    });

    $response->assertJsonFragment(['name' => 'Taylor Otwell']);

<a name="assert-json-is-array"></a>
#### assertJsonIsArray

Утверждает, что ответ JSON представляет собой массив:

    $response->assertJsonIsArray();

<a name="assert-json-is-object"></a>
#### assertJsonIsObject

Утверждает, что ответ JSON представляет собой объект:

    $response->assertJsonIsObject();

<a name="assert-json-missing"></a>
#### assertJsonMissing

Утверждает, что ответ не содержит указанных данных JSON:

    $response->assertJsonMissing(array $data);

<a name="assert-json-missing-exact"></a>
#### assertJsonMissingExact

Утверждает, что ответ не содержит точных указанных данных JSON:

    $response->assertJsonMissingExact(array $data);

<a name="assert-json-missing-validation-errors"></a>
#### assertJsonMissingValidationErrors

Утверждает, что ответ не содержит ошибок валидации JSON для переданных ключей:

    $response->assertJsonMissingValidationErrors($keys);

> [!NOTE]
> Более общий метод [assertValid](#assert-valid) может использоваться для подтверждения того, что в ответе нет ошибок проверки, которые были возвращены как JSON **и** что ошибки не были записаны в хранилище сеанса.

<a name="assert-json-path"></a>
#### assertJsonPath

Утверждает, что ответ содержит конкретные данные по указанному пути:

    $response->assertJsonPath($path, $expectedValue);

Например, если ваше приложение возвращает следующий ответ JSON:

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

Вы можете утверждать, что свойство `name` объекта `user` соответствует переданному значению следующим образом:

    $response->assertJsonPath('user.name', 'Steve Schoger');

<a name="assert-json-missing-path"></a>
#### assertJsonMissingPath

Утверждает, что ответ не содержит указанного пути:

    $response->assertJsonMissingPath($path);

Например, если ваше приложение возвращает следующий ответ JSON:

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

Вы можете утверждать, что ответ не содержит свойства `email` объекта `user`:

    $response->assertJsonMissingPath('user.email');

<a name="assert-json-structure"></a>
#### assertJsonStructure

Утверждает, что ответ имеет переданную структуру JSON:

    $response->assertJsonStructure(array $structure);

Например, если ответ JSON, возвращаемый вашим приложением, содержит следующие данные:

```json
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

Вы можете утверждать, что структура JSON соответствует вашим ожиданиям, например:

    $response->assertJsonStructure([
        'user' => [
            'name',
        ]
    ]);

Иногда ответы JSON, возвращаемые вашим приложением, могут содержать массивы объектов:

```json
{
    "user": [
        {
            "name": "Steve Schoger",
            "age": 55,
            "location": "Earth"
        },
        {
            "name": "Mary Schoger",
            "age": 60,
            "location": "Earth"
        }
    ]
}
```

В этой ситуации вы можете использовать символ `*` для утверждения о структуре всех объектов в массиве:

    $response->assertJsonStructure([
        'user' => [
            '*' => [
                 'name',
                 'age',
                 'location'
            ]
        ]
    ]);

<a name="assert-json-validation-errors"></a>
#### assertJsonValidationErrors

Утверждает, что ответ содержит переданные ошибки валидации JSON для переданных ключей. Этот метод следует использовать при утверждении ответов, в которых ошибки валидации возвращаются как структура JSON, а не кратковременно передаются в сессию:

    $response->assertJsonValidationErrors(array $data, $responseKey = 'errors');

> [!NOTE]
> Более общий метод [assertInvalid](#assert-invalid) может использоваться для подтверждения того, что в ответе есть ошибки проверки, возвращенные как JSON **или** что ошибки были записаны в хранилище сеанса.

<a name="assert-json-validation-error-for"></a>
#### assertJsonValidationErrorFor

Утверждает, что в ответе есть какие-либо ошибки проверки JSON для данного ключа:

    $response->assertJsonValidationErrorFor(string $key, $responseKey = 'errors');

<a name="assert-method-not-allowed"></a>
#### assertMethodNotAllowed

Утверждает, что ответ имеет код `405` состояния HTTP – `method not allowed`:

    $response->assertMethodNotAllowed();

<a name="assert-moved-permanently"></a>
#### assertMovedPermanently

Утверждает, что ответ имеет код `301` состояния HTTP – `moved permanently`:

    $response->assertMovedPermanently();

<a name="assert-location"></a>
#### assertLocation

Утверждает, что ответ имеет переданное значение URI в заголовке `Location`:

    $response->assertLocation($uri);

<a name="assert-content"></a>
#### assertContent

Утверждает, что указанная строка соответствует содержимому ответа:

    $response->assertContent($value);

<a name="assert-no-content"></a>
#### assertNoContent

Утверждает, что ответ имеет код `204` состояния HTTP – `no content`:

    $response->assertNoContent($status = 204);

<a name="assert-streamed-content"></a>
#### assertStreamedContent

Утверждает, что указанная строка соответствует потоковому содержимому ответа:

    $response->assertStreamedContent($value);

<a name="assert-not-found"></a>
#### assertNotFound

Утверждает, что ответ имеет код `404` состояния HTTP – `not found`:

    $response->assertNotFound();

<a name="assert-ok"></a>
#### assertOk

Утверждает, что ответ имеет код `200` состояния HTTP – `OK`:

    $response->assertOk();

<a name="assert-payment-required"></a>
#### assertPaymentRequired

Утверждает, что ответ имеет код `402` состояния HTTP – `payment required`:

    $response->assertPaymentRequired();

<a name="assert-plain-cookie"></a>
#### assertPlainCookie

Утверждает, что ответ содержит переданный незашифрованный cookie:

    $response->assertPlainCookie($cookieName, $value = null);

<a name="assert-redirect"></a>
#### assertRedirect

Утверждает, что ответ является перенаправлением на указанный URI:

    $response->assertRedirect($uri = null);

<a name="assert-redirect-contains"></a>
#### assertRedirectContains

Утверждает, перенаправляет ли ответ на URI, который содержит данную строку:

    $response->assertRedirectContains($string);

<a name="assert-redirect-to-route"></a>
#### assertRedirectToRoute

Утвердите, что ответ представляет собой перенаправление на указанный [именованный маршрут](/docs/{{version}}/routing#named-routes):

    $response->assertRedirectToRoute($name, $parameters = []);

<a name="assert-redirect-to-signed-route"></a>
#### assertRedirectToSignedRoute

Утвердите, что ответ представляет собой перенаправление на указанный [подписанный маршрут](/docs/{{version}}/urls#signed-urls)::

    $response->assertRedirectToSignedRoute($name = null, $parameters = []);

<a name="assert-request-timeout"></a>
#### assertRequestTimeout

Утверждает, что ответ имеет код `408` состояния HTTP – `request timeout`:

    $response->assertRequestTimeout();

<a name="assert-see"></a>
#### assertSee

Утверждает, что переданная строка содержится в ответе. Это утверждение автоматически экранирует переданную строку, если вы не передадите второй аргумент как `false`:

    $response->assertSee($value, $escaped = true);

<a name="assert-see-in-order"></a>
#### assertSeeInOrder

Утверждает, что переданные строки содержатся в ответе в указанном порядке. Это утверждение автоматически экранирует переданные строки, если вы не передадите второй аргумент как `false`:

    $response->assertSeeInOrder(array $values, $escaped = true);

<a name="assert-see-text"></a>
#### assertSeeText

Утверждает, что переданная строка содержится в тексте ответа. Это утверждение автоматически экранирует переданную строку, если вы не передадите второй аргумент как `false`. Этот метод передаст содержимое ответа PHP-функции `strip_tags` перед тем, как выполнить утверждение:

    $response->assertSeeText($value, $escaped = true);

<a name="assert-see-text-in-order"></a>
#### assertSeeTextInOrder

Утверждает, что переданные строки содержатся в тексте ответа в указанном порядке. Это утверждение автоматически экранирует переданные строки, если вы не передадите второй аргумент как `false`. Этот метод передаст содержимое ответа PHP-функции `strip_tags` перед тем, как выполнить утверждение:

    $response->assertSeeTextInOrder(array $values, $escaped = true);

<a name="assert-server-error"></a>
#### assertServerError

Утверждает, что ответ имеет код состояния HTTP соответствующий ошибке сервера - `>= 500 , < 600` :

    $response->assertServerError();

<a name="assert-server-unavailable"></a>
#### assertServiceUnavailable

Утверждает, что ответ имеет код `503` состояния HTTP – `Service Unavailable`:

    $response->assertServiceUnavailable();

<a name="assert-session-has"></a>
#### assertSessionHas

Утверждает, что сессия содержит переданный фрагмент данных:

    $response->assertSessionHas($key, $value = null);

Если необходимо, замыкание может быть предоставлено в качестве второго аргумента метода `assertSessionHas`. Утверждение пройдет, если замыкание вернет `true`:

    $response->assertSessionHas($key, function (User $value) {
        return $value->name === 'Taylor Otwell';
    });

<a name="assert-session-has-input"></a>
#### assertSessionHasInput

Утверждает, что сессия имеет переданное значение в [массиве входящих данных кратковременного сохранения](responses#redirecting-with-flashed-session-data):

    $response->assertSessionHasInput($key, $value = null);

Если необходимо, замыкание может быть предоставлено в качестве второго аргумента метода `assertSessionHasInput`. Утверждение пройдет, если замыкание вернет `true`:

    use Illuminate\Support\Facades\Crypt;

    $response->assertSessionHasInput($key, function (string $value) {
        return Crypt::decryptString($value) === 'secret';
    });

<a name="assert-session-has-all"></a>
#### assertSessionHasAll

Утверждает, что сессия содержит переданный массив пар ключ / значение:

    $response->assertSessionHasAll(array $data);

Например, если сессия вашего приложения содержит ключи `name` и `status`, вы можете утверждать, что оба они существуют и имеют указанные значения, например:

    $response->assertSessionHasAll([
        'name' => 'Taylor Otwell',
        'status' => 'active',
    ]);

<a name="assert-session-has-errors"></a>
#### assertSessionHasErrors

Утверждает, что сессия содержит ошибку для переданных `$keys`. Если `$keys` является ассоциативным массивом, следует утверждать, что сессия содержит конкретное сообщение об ошибке (значение) для каждого поля (ключа). Этот метод следует использовать при тестировании маршрутов, которые передают ошибки валидации в сессию вместо того, чтобы возвращать их в виде структуры JSON:

    $response->assertSessionHasErrors(
        array $keys = [], $format = null, $errorBag = 'default'
    );

Например, чтобы утверждать, что поля `name` и `email` содержат сообщения об ошибках валидации, которые были переданы в сессию, вы можете вызвать метод `assertSessionHasErrors` следующим образом:

    $response->assertSessionHasErrors(['name', 'email']);

Или вы можете утверждать, что переданное поле имеет конкретное сообщение об ошибке валидации:

    $response->assertSessionHasErrors([
        'name' => 'The given name was invalid.'
    ]);

> [!NOTE]
> Более общий метод [assertInvalid](#assert-invalid) может быть использован для проверки, что ответ содержит ошибки валидации, представленные в формате JSON **или** что ошибки были сохранены в хранилище сессий.

<a name="assert-session-has-errors-in"></a>
#### assertSessionHasErrorsIn

Утверждает, что сессия содержит ошибку для переданных `$keys` в конкретной [коллекции ошибок](/docs/{{version}}/validation#named-error-bags). Если `$keys` является ассоциативным массивом, убедитесь, что сессия содержит конкретное сообщение об ошибке (значение) для каждого поля (ключа) в коллекции ошибок:

    $response->assertSessionHasErrorsIn($errorBag, $keys = [], $format = null);

<a name="assert-session-has-no-errors"></a>
#### assertSessionHasNoErrors

Утверждает, что в сессии нет ошибок валидации:

    $response->assertSessionHasNoErrors();

<a name="assert-session-doesnt-have-errors"></a>
#### assertSessionDoesntHaveErrors

Утверждает, что в сессии нет ошибок валидации для переданных ключей:

    $response->assertSessionDoesntHaveErrors($keys = [], $format = null, $errorBag = 'default');

> [!NOTE]
> Более общий метод [assertValid](#assert-valid)  может быть использован для проверки того, что ответ не содержит ошибок валидации, представленных в формате JSON **и** что ошибок не было сохранено в хранилище сессий.

<a name="assert-session-missing"></a>
#### assertSessionMissing

Утверждает, что сессия не содержит переданного ключа:

    $response->assertSessionMissing($key);

<a name="assert-status"></a>
#### assertStatus

Утверждает, что ответ имеет указанный код `$code` состояния HTTP:

    $response->assertStatus($code);

<a name="assert-successful"></a>
#### assertSuccessful

Утверждает, что ответ имеет код `>= 200` и `< 300` состояния HTTP – `successful`:

    $response->assertSuccessful();

<a name="assert-too-many-requests"></a>
#### assertTooManyRequests

Утверждает, что ответ имеет код `429` состояния HTTP – `too many requests`:

    $response->assertTooManyRequests();

<a name="assert-unauthorized"></a>
#### assertUnauthorized

Утверждает, что ответ имеет код `401` состояния HTTP – `unauthorized`:

    $response->assertUnauthorized();

<a name="assert-unprocessable"></a>
#### assertUnprocessable

Утверждает, что ответ имеет необработанный код `422` состояния HTTP:

    $response->assertUnprocessable();

<a name="assert-unsupported-media-type"></a>
#### assertUnsupportedMediaType

Утверждает, что ответ имеет код `415` состояния HTTP – `unsupported media type`:

    $response->assertUnsupportedMediaType();

<a name="assert-valid"></a>
#### assertValid

Утверждает, что в ответе нет ошибок валидации для заданных ключей. Этот метод можно использовать для утверждения против ответов, в которых ошибки проверки возвращаются в виде структуры JSON или ошибки проверки были переданы в сессию:

    // Утверждает, что ошибок проверки нет...
    $response->assertValid();

    // Утверждает, что данные ключи не имеют ошибок проверки...
    $response->assertValid(['name', 'email']);

<a name="assert-invalid"></a>
#### assertInvalid

Утверждает, что в ответе есть ошибки валидации для заданных ключей. Этот метод можно использовать для утверждения против ответов, где ошибки проверки возвращаются в виде структуры JSON или где ошибки проверки были переданы в сессию:

    $response->assertInvalid(['name', 'email']);

Вы также можете утверждать, что данный ключ имеет определенное сообщение об ошибке валидации. При этом вы можете предоставить все сообщение или только небольшую его часть:

    $response->assertInvalid([
        'name' => 'The name field is required.',
        'email' => 'valid email address',
    ]);

<a name="assert-view-has"></a>
#### assertViewHas

Утверждает, что шаблон ответа содержит переданный фрагмент данных:

    $response->assertViewHas($key, $value = null);

Передача закрытия в качестве второго аргумента методу `assertViewHas` позволит вам проверять и делать утверждения в отношении определенного фрагмента данных представления:

    $response->assertViewHas('user', function (User $user) {
        return $user->name === 'Taylor';
    });

Кроме того, данные шаблона могут быть доступны как переменные массива в ответе, что позволяет вам удобно инспектировать их:

```php tab=Pest
expect($response['name'])->toBe('Taylor');
```

```php tab=PHPUnit
$this->assertEquals('Taylor', $response['name']);
```

<a name="assert-view-has-all"></a>
#### assertViewHasAll

Утверждает, что шаблон ответа содержит переданный список данных:

    $response->assertViewHasAll(array $data);

Этот метод может использоваться, чтобы утверждать, что шаблон просто содержит данные с соответствующими переданными ключами:

    $response->assertViewHasAll([
        'name',
        'email',
    ]);

Или вы можете утверждать, что данные шаблона присутствуют и имеют определенные значения:

    $response->assertViewHasAll([
        'name' => 'Taylor Otwell',
        'email' => 'taylor@example.com,',
    ]);

<a name="assert-view-is"></a>
#### assertViewIs

Утверждает, что маршрутом был возвращен указанный шаблон:

    $response->assertViewIs($value);

<a name="assert-view-missing"></a>
#### assertViewMissing

Утверждает, что переданный ключ данных не был доступен для шаблона, возвращенного ответом приложения:

    $response->assertViewMissing($key);

<a name="authentication-assertions"></a>
### Утверждения аутентификации

Laravel также содержит множество утверждений, связанных с аутентификацией, которые вы можете использовать в функциональных тестах вашего приложения. Обратите внимание, что эти методы вызываются в самом тестовом классе, а не в экземпляре `Illuminate\Testing\TestResponse`, возвращаемом такими методами, как `get` и `post`.

<a name="assert-authenticated"></a>
#### assertAuthenticated

Утверждает, что пользователь аутентифицирован:

    $this->assertAuthenticated($guard = null);

<a name="assert-guest"></a>
#### assertGuest

Утверждает, что пользователь не аутентифицирован:

    $this->assertGuest($guard = null);

<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

Утверждает, что конкретный пользователь аутентифицирован:

    $this->assertAuthenticatedAs($user, $guard = null);

<a name="validation-assertions"></a>
## Утверждения валидации

Laravel предоставляет два основных метода утверждения, связанных с валидацией, которые вы можете использовать, чтобы убедиться, что данные, предоставленные в вашем запросе, являются валидными или невалидными.

<a name="validation-assert-valid"></a>
#### assertValid

Утверждает, что ответ не содержит ошибок валидации для указанных ключей. Этот метод может использоваться для проверки ответов, где ошибки валидации представлены в виде JSON-структуры или где ошибки валидации сохраняются в сессии:

    // Утверждает, что ошибок валидации нет...
    $response->assertValid();

    // Утверждает, что нет ошибок валидации для указанных ключей...
    $response->assertValid(['name', 'email']);

<a name="validation-assert-invalid"></a>
#### assertInvalid

Утверждает, что ответ содержит ошибки валидации для указанных ключей. Этот метод может использоваться для проверки ответов, где ошибки валидации представлены в виде JSON-структуры или где ошибки валидации сохраняются в сессии:

    $response->assertInvalid(['name', 'email']);

Также вы можете утверждать, что для указанного ключа есть определенное сообщение об ошибке валидации. При этом вы можете предоставить либо полное сообщение, либо только его небольшую часть:

    $response->assertInvalid([
        'name' => 'The name field is required.',
        'email' => 'valid email address',
    ]);
