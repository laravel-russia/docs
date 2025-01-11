---
git: 24a9f991e9a7aa18f32e4ed48a5b2abd6ab0b99d
---

# Тестирование · Тесты консольных команд

<a name="introduction"></a>
## Введение

Помимо упрощенного HTTP-тестирования, Laravel предлагает простой API для тестирования [пользовательских консольных команд](artisan) вашего приложения.

<a name="success-failure-expectations"></a>
## Ожидания успеха / неудачи

Для начала давайте рассмотрим, как делать утверждения относительно кода выхода команды Artisan. Для этого мы будем использовать метод `artisan` для вызова Artisan-команды из нашего теста. Затем мы будем использовать метод `assertExitCode`, чтобы подтвердить, что команда завершилась с заданным кодом выхода:

```php tab=Pest
test('console command', function () {
    $this->artisan('inspire')->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('inspire')->assertExitCode(0);
}
```

Вы можете использовать метод `assertNotExitCode` чтобы подтвердить, что команда не завершилась с заданным кодом выхода:

    $this->artisan('inspire')->assertNotExitCode(1);

Конечно, все команды терминала обычно завершаются с кодом состояния `0`, когда они успешны, и с ненулевым кодом выхода, когда они не успешны. Поэтому для удобства вы можете использовать утверждения `assertSuccessful` и `assertFailed` чтобы утверждать, что данная команда завершилась с успешным кодом выхода или нет:

    $this->artisan('inspire')->assertSuccessful();

    $this->artisan('inspire')->assertFailed();

<a name="input-output-expectations"></a>
## Ожидания ввода / вывода

Laravel позволяет вам легко «имитировать» ввод пользователем в консольных командах, используя метод `expectsQuestion`. Кроме того, вы можете указать код выхода / возврата и текст, который вы ожидаете получить от консольной команды, используя методы `assertExitCode` и `expectsOutput`. Например, рассмотрим следующую консольную команду:

    Artisan::command('question', function () {
        $name = $this->ask('What is your name?');

        $language = $this->choice('Which language do you prefer?', [
            'PHP',
            'Ruby',
            'Python',
        ]);

        $this->line('Your name is '.$name.' and you prefer '.$language.'.');
    });

Вы можете проверить эту команду с помощью следующего теста:

```php tab=Pest
test('console command', function () {
    $this->artisan('question')
         ->expectsQuestion('What is your name?', 'Taylor Otwell')
         ->expectsQuestion('Which language do you prefer?', 'PHP')
         ->expectsOutput('Your name is Taylor Otwell and you prefer PHP.')
         ->doesntExpectOutput('Your name is Taylor Otwell and you prefer Ruby.')
         ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('question')
         ->expectsQuestion('What is your name?', 'Taylor Otwell')
         ->expectsQuestion('Which language do you prefer?', 'PHP')
         ->expectsOutput('Your name is Taylor Otwell and you prefer PHP.')
         ->doesntExpectOutput('Your name is Taylor Otwell and you prefer Ruby.')
         ->assertExitCode(0);
}
```

Если вы используете функции `search` или `multisearch`, предоставляемые [Laravel Prompts](/docs/{{version}}/prompts), вы можете использовать утверждение `expectsSearch`, чтобы имитировать ввод пользователя, результаты поиска и выбор:

```php tab=Pest
test('console command', function () {
    $this->artisan('example')
         ->expectsSearch('What is your name?', search: 'Tay', answers: [
            'Taylor Otwell',
            'Taylor Swift',
            'Darian Taylor'
         ], answer: 'Taylor Otwell')
         ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('example')
         ->expectsSearch('What is your name?', search: 'Tay', answers: [
            'Taylor Otwell',
            'Taylor Swift',
            'Darian Taylor'
         ], answer: 'Taylor Otwell')
         ->assertExitCode(0);
}
```

Вы также можете утверждать, что консольная команда не генерирует никакого вывода, используя метод `doesntExpectOutput`:

```php tab=Pest
test('console command', function () {
    $this->artisan('example')
         ->doesntExpectOutput()
         ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('example')
            ->doesntExpectOutput()
            ->assertExitCode(0);
}
```

Методы `expectsOutputToContain` и `doesntExpectOutputToContain` могут использоваться для создания утверждений относительно части вывода:

```php tab=Pest
test('console command', function () {
    $this->artisan('example')
         ->expectsOutputToContain('Taylor')
         ->assertExitCode(0);
});
```

```php tab=PHPUnit
/**
 * Test a console command.
 */
public function test_console_command(): void
{
    $this->artisan('example')
            ->expectsOutputToContain('Taylor')
            ->assertExitCode(0);
}
```

<a name="confirmation-expectations"></a>
#### Ожидания подтверждения

При написании команды, которая ожидает подтверждения в виде ответа «да» или «нет», вы можете использовать метод `expectsConfirmation`:

    $this->artisan('module:import')
        ->expectsConfirmation('Do you really wish to run this command?', 'no')
        ->assertExitCode(1);

<a name="table-expectations"></a>
#### Таблица ожиданий

Если ваша команда отображает таблицу информации с использованием метода `table` Artisan, может быть обременительно записывать ожидаемые результаты для всей таблицы. Вместо этого вы можете использовать метод `expectsTable`. Этот метод принимает заголовки таблицы в качестве первого аргумента и данные таблицы в качестве второго аргумента:

    $this->artisan('users:all')
        ->expectsTable([
            'ID',
            'Email',
        ], [
            [1, 'taylor@example.com'],
            [2, 'abigail@example.com'],
        ]);

<a name="console-events"></a>
## События консоли

По умолчанию события `Illuminate\Console\Events\CommandStarting` и `Illuminate\Console\Events\CommandFinished` не генерируются при запуске тестов вашего приложения. Однако вы можете включить эти события для данного класса тестов, добавив трейт `Illuminate\Foundation\Testing\WithConsoleEvents` в класс:

```php tab=Pest
<?php

use Illuminate\Foundation\Testing\WithConsoleEvents;

uses(WithConsoleEvents::class);

// ...
```

```php tab=PHPUnit
<?php

namespace Tests\Feature;

use Illuminate\Foundation\Testing\WithConsoleEvents;
use Tests\TestCase;

class ConsoleEventTest extends TestCase
{
    use WithConsoleEvents;

    // ...
}
```
