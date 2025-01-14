---
git: 37a8b499fc099ea49dd232d64c6cc4fd3f7ad7f7
---

# Обработка ошибок (Exception)

<a name="introduction"></a>
## Введение

Когда вы запускаете новый проект Laravel, обработка ошибок и исключений уже настроена для вас; однако в любой момент вы можете использовать метод `withExceptions` в файле `bootstrap/app.php` вашего приложения, чтобы управлять тем, как ваше приложение сообщает об исключениях и обрабатывает их.

Объект `$exceptions`, предоставляемый замыканию `withExceptions`, является экземпляром `Illuminate\Foundation\Configuration\Exceptions` и отвечает за управление обработкой исключений в вашем приложении. В этой документации мы углубимся в этот объект.

<a name="configuration"></a>
## Конфигурирование

Параметр `debug` в конфигурационном файле `config/app.php` определяет, сколько информации об ошибке фактически отобразится пользователю. По умолчанию этот параметр установлен, чтобы учесть значение переменной окружения `APP_DEBUG`, которая содержится в вашем файле `.env`.

Во время локальной разработки вы должны установить для переменной окружения `APP_DEBUG` значение `true`. **Во время эксплуатации приложения это значение всегда должно быть `false`. Если в рабочем окружении будет установлено значение `true`, вы рискуете раскрыть конфиденциальные значения конфигурации конечным пользователям вашего приложения.**

<a name="handling-exceptions"></a>
## Обработка исключений

<a name="reporting-exceptions"></a>
### Отчет об исключениях

В Laravel отчеты об исключениях используются для регистрации исключений или их отправки во внешнюю службу [Sentry](https://github.com/getsentry/sentry-laravel) или [Flare](https://flareapp.io). По умолчанию исключения будут регистрироваться на основе вашей конфигурации [logging](/docs/{{version}}/logging). Однако вы можете регистрировать исключения по своему усмотрению.

Если вам нужно сообщать о различных типах исключений разными способами, вы можете использовать метод исключений `report` в файле `bootstrap/app.php` вашего приложения, чтобы зарегистрировать замыкание, которое должно выполняться, когда необходимо сообщить об исключении определенного типа. Laravel определит, о каком типе исключения сообщает замыкание, исследуя подсказку типа замыкания:

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        });
    })

Когда вы регистрируете собственные замыкания для создания отчетов об исключениях, используя метод `report`, Laravel по-прежнему регистрирует исключение, используя конфигурацию логирования по умолчанию для приложения. Если вы хотите остановить распространение исключения в стек журналов по умолчанию, вы можете использовать метод `stop` при определении замыкания отчета или вернуть `false` из замыкания:

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->report(function (InvalidOrderException $e) {
            // ...
        })->stop();

        $exceptions->report(function (InvalidOrderException $e) {
            return false;
        });
    })

> [!NOTE]
> Чтобы настроить отчет об исключениях для переданного исключения, вы можете рассмотреть возможность использования [отчетных исключений](#renderable-exceptions).

<a name="global-log-context"></a>
#### Глобальное содержимое журнала

Если доступно, Laravel автоматически добавляет идентификатор текущего пользователя в каждое сообщение журнала исключения в качестве контекстных данных. Вы можете определить свои собственные глобальные контекстные данные, используя метод исключения `context` в файле `bootstrap/app.php` вашего приложения. Эта информация будет включена в каждое сообщение журнала исключения, записанное вашим приложением:

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->context(fn () => [
            'foo' => 'bar',
        ]);
    })

<a name="exception-log-context"></a>
#### Контекст журнала исключений

Добавление контекста к каждому сообщению в журнале может быть полезным, но иногда у конкретного исключения может быть уникальный контекст, который вы хотели бы включить в журнал. Определив метод `context` в одном из исключений вашего приложения, вы можете указать любые данные, относящиеся к этому исключению, которые должны быть добавлены в журнал записи об исключении:

    <?php

    namespace App\Exceptions;

    use Exception;

    class InvalidOrderException extends Exception
    {
        // ...

        /**
         * Получить контекстную информацию исключения.
         *
         * @return array<string, mixed>
         */
        public function context(): array
        {
            return ['order_id' => $this->orderId];
        }
    }

<a name="the-report-helper"></a>
#### Помощник `report`

По желанию может потребоваться сообщить об исключении, но продолжить обработку текущего запроса. Помощник `report` позволяет вам быстро сообщить об исключении, не отображая страницу с ошибкой для пользователя:

    public function isValid(string $value): bool
    {
        try {
            // Проверка `$value` ...
        } catch (Throwable $e) {
            report($e);

            return false;
        }
    }

<a name="deduplicating-reported-exceptions"></a>
#### Исключения дубликатов

Если вы используете функцию `report` в вашем приложении, вы иногда можете сообщать об одном и том же исключении несколько раз, создавая дублирующие записи в журналах.

Если вы хотите, чтобы об одном экземпляре исключения сообщалось только один раз, вы можете вызвать метод исключения `dontReportDuplicates` в файле `bootstrap/app.php` вашего приложения:

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReportDuplicates();
    })

Теперь, когда функция `report` вызывается с тем же экземпляром исключения, будет сообщено только первое вызов:

```php
$original = new RuntimeException('Упс!');

report($original); // сообщено

try {
    throw $original;
} catch (Throwable $caught) {
    report($caught); // проигнорировано
}

report($original); // проигнорировано
report($caught); // проигнорировано
```

<a name="exception-log-levels"></a>
### Уровни журнала исключений

Когда сообщения записываются в [журнал вашего приложения](/docs/{{version}}/logging), сообщения записываются с указанным [уровнем журнала](/docs/{{version}}/logging#log-levels), который указывает на серьезность или важность сообщения, которое записывается.

Как отмечено выше, даже когда вы регистрируете пользовательский обратный вызов сообщения об исключении с использованием метода `report`, Laravel все равно будет записывать исключение с использованием конфигурации регистрации журнала по умолчанию для приложения. Однако поскольку уровень журнала иногда может влиять на каналы, на которых записывается сообщение, вы можете настроить уровень журнала, на котором определенные исключения записываются.

Для этого вы можете использовать метод исключения `level` в файле `bootstrap/app.php` вашего приложения. Этот метод получает тип исключения в качестве первого аргумента и уровень журнала в качестве второго аргумента:

    use PDOException;
    use Psr\Log\LogLevel;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->level(PDOException::class, LogLevel::CRITICAL);
    })

<a name="ignoring-exceptions-by-type"></a>
### Игнорирование исключений по типу

При создании приложения могут возникнуть некоторые типы исключений, о которых вы никогда не захотите сообщать. Чтобы игнорировать эти исключения, вы можете использовать метод исключений `dontReport` в файле `bootstrap/app.php` вашего приложения. Ни о каком классе, предоставленном этому методу, никогда не будет сообщено; однако они все равно могут иметь собственную логику рендеринга:

    use App\Exceptions\InvalidOrderException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->dontReport([
            InvalidOrderException::class,
        ]);
    })

В качестве альтернативы вы можете просто «пометить» класс исключений с помощью интерфейса `Illuminate\Contracts\Debug\ShouldntReport`. Когда исключение помечено этим интерфейсом, обработчик исключений Laravel никогда не сообщит об этом:

```php
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Contracts\Debug\ShouldntReport;

class PodcastProcessingException extends Exception implements ShouldntReport
{
    //
}
```

Внутри Laravel уже игнорирует некоторые типы ошибок, например исключения, возникающие из-за ошибок 404 HTTP или ответов 419 HTTP, сгенерированных недействительными токенами CSRF. Если вы хотите указать Laravel прекратить игнорировать определенный тип исключения, вы можете использовать метод исключения `stopIgnoring` в файле `bootstrap/app.php` вашего приложения:

    use Symfony\Component\HttpKernel\Exception\HttpException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->stopIgnoring(HttpException::class);
    })

<a name="rendering-exceptions"></a>
### Отображение исключений

По умолчанию обработчик исключений Laravel преобразует исключения в HTTP-ответ. Однако вы можете зарегистрировать свое замыкание для исключений заданного типа. Вы можете добиться этого, используя метод исключения `render` в файле `bootstrap/app.php` вашего приложения.

Замыкание, переданное методу `render`, должно вернуть экземпляр `Illuminate\Http\Response`, который может быть сгенерирован с помощью функции `response`. Laravel определит, какой тип исключения отображает замыкание с помощью типизации аргументов:

    use App\Exceptions\InvalidOrderException;
    use Illuminate\Http\Request;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (InvalidOrderException $e, Request $request) {
            return response()->view('errors.invalid-order', status: 500);
        });
    })

Вы также можете использовать метод `render` чтобы переопределить отображение для встроенных исключений Laravel или Symfony, таких, как `NotFoundHttpException`. Если замыкание, переданное методу `render` не возвращает значения, будет использоваться отрисовка исключений Laravel по умолчанию:

    use Illuminate\Http\Request;
    use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->render(function (NotFoundHttpException $e, Request $request) {
            if ($request->is('api/*')) {
                return response()->json([
                    'message' => 'Запись не найдена.'
                ], 404);
            }
        });
    })

<a name="rendering-exceptions-as-json"></a>
#### Отображение исключений в формате JSON

При обработке исключения Laravel автоматически определяет, должно ли исключение быть отображено в виде ответа HTML или JSON, на основе заголовка `Accept` запроса. Если вы хотите настроить, как Laravel определяет, следует ли отображать ответы об исключениях HTML или JSON, вы можете использовать метод `shouldRenderJsonWhen`:

    use Illuminate\Http\Request;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->shouldRenderJsonWhen(function (Request $request, Throwable $e) {
            if ($request->is('admin/*')) {
                return true;
            }

            return $request->expectsJson();
        });
    })

<a name="customizing-the-exception-response"></a>
#### Настройка ответа на исключение

В редких случаях вам может потребоваться настроить весь HTTP-ответ, отображаемый обработчиком исключений Laravel. Для этого вы можете зарегистрировать закрытие настройки ответа, используя метод `respond`:

    use Symfony\Component\HttpFoundation\Response;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->respond(function (Response $response) {
            if ($response->getStatusCode() === 419) {
                return back()->with([
                    'message' => 'Срок действия страницы истек, попробуйте еще раз.',
                ]);
            }

            return $response;
        });
    })

<a name="renderable-exceptions"></a>
### Отчетные и отображаемые исключения

Вместо того чтобы настраивать пользовательское поведение отчетов и отображение ошибок в файле `bootstrap/app.php` вашего приложения, вы можете определить методы `report` и `render` непосредственно в самих классах исключений вашего приложения. Когда эти методы существуют, фреймворк автоматически будет вызывать их для обработки ошибок:

    <?php

    namespace App\Exceptions;

    use Exception;
    use Illuminate\Http\Request;
    use Illuminate\Http\Response;

    class InvalidOrderException extends Exception
    {
        /**
         * Отчитаться об исключении.
         */
        public function report(): void
        {
            // ...
        }

        /**
         * Преобразовать исключение в HTTP-ответ.
         */
        public function render(Request $request): Response
        {
            return response(/* ... */);
        }
    }

Если ваше исключение расширяет исключение, которое уже доступно для визуализации, например встроенное исключение Laravel или Symfony, вы можете вернуть `false` из метода `render` исключения, чтобы отобразить HTTP-ответ исключения по умолчанию:

    /**
     * Преобразовать исключение в HTTP-ответ.
     */
    public function render(Request $request): Response|bool
    {
        if (/** Определить, требуется ли для исключения пользовательское отображение */) {

            return response(/* ... */);
        }

        return false;
    }

Если ваше исключение содержит пользовательскую логику отчетности, которая необходима только при выполнении определенных условий, то вам может потребоваться указать Laravel когда сообщать об исключении, используя конфигурацию обработки исключений по умолчанию. Для этого вы можете вернуть `false` из метода `report` исключения:

    /**
     * Сообщить об исключении.
     */
    public function report(): bool
    {
        if (/** Определить, требуется ли для исключения пользовательское отображение */) {

            // ...

            return true;
        }

        return false;
    }

> [!NOTE]
> Вы можете указать любые требуемые зависимости метода `report`, и они будут автоматически внедрены в метод [контейнером служб](/docs/{{version}}/container) Laravel.

<a name="throttling-reported-exceptions"></a>
### Ограничение на количество зарегистрированных исключений

Если ваше приложение регистрирует очень большое количество исключений, вам может потребоваться ограничить количество фактически регистрируемых или отправляемых во внешний сервис отслеживания ошибок.

Чтобы получить случайную частоту выборки исключений, вы можете использовать метод исключений `throttle` в файле `bootstrap/app.php` вашего приложения. Метод `throttle` получает замыкание, которое должно возвращать экземпляр `Lottery`:

    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            return Lottery::odds(1, 1000);
        });
    })

Также можно условно выбирать исключения на основе их типа. Если вы хотите выбирать только экземпляры конкретного класса исключений, вы можете вернуть экземпляр `Lottery` только для этого класса:

    use App\Exceptions\ApiMonitoringException;
    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof ApiMonitoringException) {
                return Lottery::odds(1, 1000);
            }
        });
    })

Вы также можете ограничивать количество исключений, зарегистрированных или отправленных во внешний сервис отслеживания ошибок, вернув экземпляр `Limit` вместо `Lottery`. Это полезно, если вы хотите защититься от внезапных всплесков исключений, засоряющих ваши логи, например, когда сторонний сервис, используемый вашим приложением, недоступен:

    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof BroadcastException) {
                return Limit::perMinute(300);
            }
        });
    })

По умолчанию ограничения будут использовать класс исключения в качестве ключа ограничения по количеству. Вы можете настроить это, указав свой собственный ключ с помощью метода `by` на `Limit`:

    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            if ($e instanceof BroadcastException) {
                return Limit::perMinute(300)->by($e->getMessage());
            }
        });
    })

Конечно же, вы можете возвращать смешанные экземпляры `Lottery` и `Limit` для разных исключений:

    use App\Exceptions\ApiMonitoringException;
    use Illuminate\Broadcasting\BroadcastException;
    use Illuminate\Cache\RateLimiting\Limit;
    use Illuminate\Support\Lottery;
    use Throwable;

    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->throttle(function (Throwable $e) {
            return match (true) {
                $e instanceof BroadcastException => Limit::perMinute(300),
                $e instanceof ApiMonitoringException => Lottery::odds(1, 1000),
                default => Limit::none(),
            };
        });
    })

<a name="http-exceptions"></a>
## HTTP-исключения

Некоторые исключения описывают коды HTTP-ошибок с сервера. Например, это может быть ошибка «страница не найдена» (404), «неавторизованный доступ» (401) или даже ошибка 500, сгенерированная разработчиком. Чтобы создать такой ответ из любой точки вашего приложения, вы можете использовать глобальный помощник `abort`:

    abort(404);

<a name="custom-http-error-pages"></a>
### Пользовательские страницы для HTTP ошибок

Laravel позволяет легко отображать пользовательские страницы ошибок для различных кодов состояния HTTP. Например, если вы хотите настроить страницу ошибок для кодов HTTP-состояния 404, создайте файл `resources/views/errors/404.blade.php`. Это представление будет отображено для всех ошибок 404, сгенерированных вашим приложением. Шаблоны в этом каталоге должны быть названы в соответствии с кодом состояния HTTP, которому они соответствуют. Экземпляр `Symfony\Component\HttpKernel\Exception\HttpException`, вызванный функцией `abort`, будет передан в шаблон как переменная `$exception`:

    <h2>{{ $exception->getMessage() }}</h2>

Вы можете опубликовать стандартные шаблоны страниц ошибок Laravel с помощью команды `vendor:publish` Artisan. После публикации шаблонов вы можете настроить их по своему вкусу:

```shell
php artisan vendor:publish --tag=laravel-errors
```

<a name="fallback-http-error-pages"></a>
#### Запасные страницы для HTTP ошибок

Вы также можете определить "запасную" страницу ошибки для определенного набора кодов состояния HTTP. Эта страница будет отображаться, если нет соответствующей страницы для конкретного кода состояния HTTP, который произошел. Для этого определите шаблон `4xx.blade.php` и шаблон `5xx.blade.php` в директории `resources/views/errors` вашего приложения.

При определении "запасных" страниц ошибок, "запасные" страницы не влияют на ответы об ошибках `404`, `500` и `503`, поскольку в Laravel есть внутренние, выделенные страницы для этих кодов состояния. Чтобы настроить страницы, отображаемые для этих кодов состояния, необходимо определить собственную страницу ошибок для каждого из них индивидуально.
