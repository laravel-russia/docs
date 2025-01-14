---
git: 137c84001cb546324e63327cb0c08ca67d3cd5fc
---

# Контекст

<a name="introduction"></a>
## Введение

«Контекстные» возможности Laravel позволяют вам собирать, извлекать и обмениваться информацией в запросах, заданиях и командах, выполняющихся в вашем приложении. Эта собранная информация также включается в журналы, записываемые вашим приложением, что дает вам более глубокое представление об истории выполнения окружающего кода, произошедшей до того, как была записана запись в журнале, и позволяет отслеживать потоки выполнения во всей распределенной системе.

<a name="how-it-works"></a>
### Как это работает

Лучший способ понять контекстные возможности Laravel — увидеть его в действии, используя встроенные функции ведения журнала. Для начала вы можете [добавить информацию в контекст](#capturing-context), используя фасад `Context`. В этом примере мы будем использовать [посредника](/docs/{{version}}/middleware) для добавления URL-адреса запроса и уникального идентификатора трассировки в контекст каждого входящего запроса:

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class AddContext
{
    /**
     * Обработка входящего запроса.
     */
    public function handle(Request $request, Closure $next): Response
    {
        Context::add('url', $request->url());
        Context::add('trace_id', Str::uuid()->toString());

        return $next($request);
    }
}
```

Информация, добавленная в контекст, автоматически добавляется в виде метаданных ко всем [записям журнала](/docs/{{version}}/logging), которые записываются на протяжении всего запроса. Добавление контекста в виде метаданных позволяет отличать информацию, передаваемую в отдельные записи журнала, от информации, передаваемой через `Context`. Например, представьте, что мы пишем следующую запись в журнале:

```php
Log::info('Пользователь прошел аутентификацию.', ['auth_id' => Auth::id()]);
```

Запись в журнале будет содержать переданный `auth_id`, но запись также будет содержать `url` и `trace_id` контекста в качестве метаданных:

```
Пользователь прошел аутентификацию. {"auth_id":27} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

Информация, добавленная в контекст, также становится доступной для заданий, отправленных в очередь. Например, представьте, что мы отправляем задание `ProcessPodcast` в очередь после добавления некоторой информации в контекст:

```php
// В нашем посреднике...
Context::add('url', $request->url());
Context::add('trace_id', Str::uuid()->toString());

// В нашем контроллере...
ProcessPodcast::dispatch($podcast);
```

При отправке задания любая информация, хранящаяся в данный момент в контексте, фиксируется и передается заданию. Собранная информация затем возвращается в текущий контекст во время выполнения задания. Итак, если бы метод `handle` нашего задания заключался в записи в журнал:

```php
class ProcessPodcast implements ShouldQueue
{
    use Queueable;

    // ...

    /**
     * Выполнение задания.
     */
    public function handle(): void
    {
        Log::info('Обработка подкаста.', [
            'podcast_id' => $this->podcast->id,
        ]);

        // ...
    }
}
```

Результирующая запись журнала будет содержать информацию, которая была добавлена ​​в контекст во время запроса, который первоначально отправил задание:

```
Обработка подкаста. {"podcast_id":95} {"url":"https://example.com/login","trace_id":"e04e1a11-e75c-4db3-b5b5-cfef4ef56697"}
```

Хотя мы сосредоточились на встроенных функциях контекста Laravel, связанных с ведением журнала, следующая документация покажет, как контекст позволяет вам обмениваться информацией через границу HTTP-запроса/задания в очереди и даже как добавлять [данные скрытого контекста](#hidden- контекст), который не записывается в записи журнала.

<a name="capturing-context"></a>
## Захват контекста

Вы можете хранить информацию в текущем контексте, используя метод `add` фасада `Context`:

```php
use Illuminate\Support\Facades\Context;

Context::add('key', 'value');
```

Чтобы добавить несколько элементов одновременно, вы можете передать ассоциативный массив методу `add`:

```php
Context::add([
    'first_key' => 'value',
    'second_key' => 'value',
]);
```

Метод `add` переопределит любое существующее значение, имеющее тот же ключ. Если вы хотите добавить информацию в контекст только в том случае, если ключ еще не существует, вы можете использовать метод `addIf`:

```php
Context::add('key', 'first');

Context::get('key');
// "first"

Context::addIf('key', 'second');

Context::get('key');
// "first"
```

<a name="conditional-context"></a>
#### Условный контекст

Метод `when` можно использовать для добавления данных в контекст на основе заданного условия. Первое замыкание, предоставленное методу `when`, будет вызвано, если данное условие оценивается как `true`, а второе замыкание будет вызвано, если условие оценивается как `false`:

```php
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Context;

Context::when(
    Auth::user()->isAdmin(),
    fn ($context) => $context->add('permissions', Auth::user()->permissions),
    fn ($context) => $context->add('permissions', []),
);
```

<a name="stacks"></a>
### Стеки

Контекст предлагает возможность создавать «стеки», которые представляют собой списки данных, хранящихся в том порядке, в котором они были добавлены. Вы можете добавить информацию в стек, вызвав метод `push`:

```php
use Illuminate\Support\Facades\Context;

Context::push('breadcrumbs', 'first_value');

Context::push('breadcrumbs', 'second_value', 'third_value');

Context::get('breadcrumbs');
// [
//     'first_value',
//     'second_value',
//     'third_value',
// ]
```

Стеки могут быть полезны для сбора исторической информации о запросе, например событий, происходящих в вашем приложении. Например, вы можете создать прослушиватель событий, который будет помещать в стек каждый раз при выполнении запроса, фиксируя SQL-запрос и его продолжительность в виде кортежа:

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Facades\DB;

DB::listen(function ($event) {
    Context::push('queries', [$event->time, $event->sql]);
});
```

Вы можете определить, находится ли значение в стеке, используя методы `stackContains` и `hiddenStackContains`:

```php
if (Context::stackContains('breadcrumbs', 'first_value')) {
    //
}

if (Context::hiddenStackContains('secrets', 'first_value')) {
    //
}
```

Методы `stackContains` и `hiddenStackContains` также принимают замыкание в качестве второго аргумента, что позволяет лучше контролировать операцию сравнения значений:

```php
use Illuminate\Support\Facades\Context;
use Illuminate\Support\Str;

return Context::stackContains('breadcrumbs', function ($value) {
    return Str::startsWith($value, 'query_');
});
```

<a name="retrieving-context"></a>
## Получение контекста

Вы можете получить информацию из контекста, используя метод `get` фасада `Context`:

```php
use Illuminate\Support\Facades\Context;

$value = Context::get('key');
```

Метод `only` можно использовать для получения подмножества информации в контексте:

```php
$data = Context::only(['first_key', 'second_key']);
```

Метод `pull` можно использовать для извлечения информации из контекста и немедленного удаления ее из контекста:

```php
$value = Context::pull('key');
```

Если контекстные данные хранятся в [стеке](#stacks), вы можете извлекать элементы из стека, используя метод `pop`:

```php
Context::push('breadcrumbs', 'first_value', 'second_value');

Context::pop('breadcrumbs')
// second_value

Context::get('breadcrumbs');
// ['first_value']
```

Если вы хотите получить всю информацию, хранящуюся в контексте, вы можете вызвать метод `all`:

```php
$data = Context::all();
```

<a name="determining-item-existence"></a>
### Определение существования элемента

Вы можете использовать метод `has`, чтобы определить, имеет ли контекст какое-либо значение, сохраненное для данного ключа:

```php
use Illuminate\Support\Facades\Context;

if (Context::has('key')) {
    // ...
}
```

Метод `has` вернет `true` независимо от сохраненного значения. Так, например, ключ со значением `null` будет считаться присутствующим:

```php
Context::add('key', null);

Context::has('key');
// true
```

<a name="removing-context"></a>
## Удаление контекста

Метод `forget` можно использовать для удаления ключа и его значения из текущего контекста:

```php
use Illuminate\Support\Facades\Context;

Context::add(['first_key' => 1, 'second_key' => 2]);

Context::forget('first_key');

Context::all();

// ['second_key' => 2]
```

Вы можете забыть несколько ключей одновременно, предоставив массив методу `forget`:

```php
Context::forget(['first_key', 'second_key']);
```

<a name="hidden-context"></a>
## Скрытый контекст

Контекст предлагает возможность хранить «скрытые» данные. Эта скрытая информация не добавляется в журналы и недоступна с помощью описанных выше методов получения данных. Контекст предоставляет другой набор методов для взаимодействия со скрытой контекстной информацией:

```php
use Illuminate\Support\Facades\Context;

Context::addHidden('key', 'value');

Context::getHidden('key');
// 'value'

Context::get('key');
// null
```

«Скрытые» методы отражают функциональность нескрытых методов, описанных выше:

```php
Context::addHidden(/* ... */);
Context::addHiddenIf(/* ... */);
Context::pushHidden(/* ... */);
Context::getHidden(/* ... */);
Context::pullHidden(/* ... */);
Context::popHidden(/* ... */);
Context::onlyHidden(/* ... */);
Context::allHidden(/* ... */);
Context::hasHidden(/* ... */);
Context::forgetHidden(/* ... */);
```

<a name="events"></a>
## События

Контекст отправляет два события, которые позволяют вам подключиться к процессу гидратации и обезвоживания контекста.

Чтобы проиллюстрировать, как можно использовать эти события, представьте, что в промежуточном программном обеспечении вашего приложения вы устанавливаете значение конфигурации `app.locale` на основе заголовка `Accept-Language` входящего HTTP-запроса. События контекста позволяют вам захватить это значение во время запроса и восстановить его в очереди, гарантируя, что отправляемые в очередь уведомления имеют правильное значение `app.locale`. Для достижения этой цели мы можем использовать события контекста и данные [hidden](#hidden-context), что будет показано в следующей документации.

<a name="dehydrating"></a>
### Обезвоживание

Всякий раз, когда задание отправляется в очередь, данные в контексте «обезвоживаются» и фиксируются вместе с полезной нагрузкой задания. Метод `Context::dehydrating` позволяет вам зарегистрировать замыкание, которое будет вызываться во время процесса обезвоживания. В рамках этого закрытия вы можете вносить изменения в данные, которые будут доступны для задания в очереди.

Обычно вам следует зарегистрировать `dehydrating` обратные вызовы в методе `boot` класса `AppServiceProvider` вашего приложения:

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * Загрузка любых сервисов приложения.
 */
public function boot(): void
{
    Context::dehydrating(function (Repository $context) {
        $context->addHidden('locale', Config::get('app.locale'));
    });
}
```

> [!NOTE]
> Не следует использовать фасад `Context` в обратном вызове `dehydrating`, так как это изменит контекст текущего процесса. Убедитесь, что вы вносите изменения только в репозиторий, переданный в обратный вызов.

<a name="hydrated"></a>
### Гидратация

Всякий раз, когда поставленное в очередь задание начинает выполняться в очереди, любой контекст, который был общим с заданием, будет «гидратирован» обратно в текущий контекст. Метод `Context::hydrated` позволяет вам зарегистрировать замыкание, которое будет вызываться во время процесса гидратации.

Обычно вам следует регистрировать `hydrated` обратные вызовы в методе `boot` класса `AppServiceProvider` вашего приложения:

```php
use Illuminate\Log\Context\Repository;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Context;

/**
 * Загрузка любых сервисов приложения.
 */
public function boot(): void
{
    Context::hydrated(function (Repository $context) {
        if ($context->hasHidden('locale')) {
            Config::set('app.locale', $context->getHidden('locale'));
        }
    });
}
```

> [!NOTE]
> Не следует использовать фасад `Context` в обратном вызове `hydrated` и вместо этого убедитесь, что вы вносите изменения только в репозиторий, переданный в обратный вызов.
