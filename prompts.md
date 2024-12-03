---
git: 77db50ec005aa0e6391511eb81ff3a0e21d7e504
---

# Prompts (Подсказки)

<a name="introduction"></a>
## Введение

[Laravel Prompts](https://github.com/laravel/prompts) - это PHP-пакет, который позволяет добавлять красивые и удобные формы в ваши приложения командной строки, с функциями, подобными браузеру, включая плейсхолдеры и валидацию.

<img src="https://laravel.com/img/docs/prompts-example.png">

Laravel Prompts идеально подходит для приема пользовательского ввода в ваших [командах Artisan консоли](/docs/{{version}}/artisan#writing-commands), но его также можно использовать в любом проекте с командной строкой на PHP.

> [!NOTE]
> Laravel Prompts поддерживает macOS, Linux и Windows с WSL. Для получения дополнительной информации, пожалуйста, ознакомьтесь с нашей документацией по [не поддерживаемым средам и резервным вариантам](#fallbacks).

<a name="installation"></a>
## Установка

Laravel Prompts уже включен в последний релиз Laravel.

Вы также можете установить Laravel Prompts в другие проекты PHP, используя менеджер пакетов Composer:

```shell
composer require laravel/prompts
```

<a name="available-prompts"></a>
## Доступные Prompts

<a name="text"></a>
### Текст

Функция `text` предложит пользователю указанный вопрос, примет введённый текст и затем вернет его:

```php
use function Laravel\Prompts\text;

$name = text('What is your name?');
```

Вы также можете добавить плейсхолдер, значение по умолчанию и информационную подсказку:

```php
$name = text(
    label: 'What is your name?',
    placeholder: 'E.g. Taylor Otwell',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

<a name="text-required"></a>
#### Обязательные значения

Если вам необходимо, чтобы было введено значение, вы можете передать аргумент `required`:

```php
$name = text(
    label: 'What is your name?',
    required: true
);
```

Если вы хотите настроить сообщение об ошибке валидации, вы также можете передать строку:

```php
$name = text(
    label: 'What is your name?',
    required: 'Your name is required.'
);
```

<a name="text-validation"></a>
#### Дополнительная Валидация

Наконец, если вы хотите выполнить дополнительную логику валидации, вы можете передать замыкание в аргумент `validate`:

```php
$name = text(
    label: 'What is your name?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

Замыкание получит введенное значение и может вернуть сообщение об ошибке или `null`, если валидация прошла успешно.

Альтернативно вы можете использовать возможности [валидатора](/docs/{{version}}/validation) Laravel. Для этого предоставьте в аргумент `validate` массив, содержащий имя атрибута и желаемые правила проверки:

```php
$name = text(
    label: 'What is your name?',
    validate: ['name' => 'required|max:255|unique:users']
);
```

<a name="textarea"></a>
### Textarea

Функция `textarea` предложит пользователю задать заданный вопрос, примет его ввод через многострочную текстовую область, а затем вернет его:

```php
use function Laravel\Prompts\textarea;

$story = textarea('Tell me a story.');
```

Вы также можете включить текст-заполнитель, значение по умолчанию и информационную подсказку:

```php
$story = textarea(
    label: 'Tell me a story.',
    placeholder: 'This is a story about...',
    hint: 'This will be displayed on your profile.'
);
```

<a name="textarea-required"></a>
#### Обязательные значения

Если вам требуется, чтобы значение было введено, то вы можете передать аргумент `required`:

```php
$story = textarea(
    label: 'Tell me a story.',
    required: true
);
```

Если вы хотите настроить сообщение проверки, вы также можете передать строку:

```php
$story = textarea(
    label: 'Tell me a story.',
    required: 'A story is required.'
);
```

<a name="textarea-validation"></a>
#### Дополнительная проверка

Наконец, если вы хотите выполнить дополнительную логику проверки, вы можете передать замыкание аргументу `validate`:

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: fn (string $value) => match (true) {
        strlen($value) < 250 => 'The story must be at least 250 characters.',
        strlen($value) > 10000 => 'The story must not exceed 10,000 characters.',
        default => null
    }
);
```

Замыкание получит введенное значение и может вернуть сообщение об ошибке или значение `null`, если проверка пройдет успешно.

Альтернативно вы можете использовать возможности [валидатора](/docs/{{version}}/validation) Laravel. Для этого предоставьте в аргумент `validate` массив, содержащий имя атрибута и желаемые правила проверки:

```php
$story = textarea(
    label: 'Tell me a story.',
    validate: ['story' => 'required|max:10000']
);
```

<a name="password"></a>
### Пароль

Функция `password` аналогична функции `text`, но ввод пользователя будет маскироваться при вводе в консоли. Это полезно при запросе чувствительной информации, такой как пароли:

```php
use function Laravel\Prompts\password;

$password = password('What is your password?');
```

Вы также можете включить плейсхолдер и информационную подсказку:

```php
$password = password(
    label: 'What is your password?',
    placeholder: 'password',
    hint: 'Minimum 8 characters.'
);
```

<a name="password-required"></a>
#### Обязательные знаяения

Если вам необходимо, чтобы было введено значение, вы можете передать аргумент `required`:

```php
$password = password(
    label: 'What is your password?',
    required: true
);
```

Если вы хотите настроить сообщение об ошибке валидации, вы также можете передать строку:

```php
$password = password(
    label: 'What is your password?',
    required: 'The password is required.'
);
```

<a name="password-validation"></a>
#### Дополнительная Валидация

Наконец, если вы хотите выполнить дополнительную логику валидации, вы можете передать замыкание в аргумент `validate`:

```php
$password = password(
    label: 'What is your password?',
    validate: fn (string $value) => match (true) {
        strlen($value) < 8 => 'The password must be at least 8 characters.',
        default => null
    }
);
```

Замыкание получит введенное значение и может вернуть сообщение об ошибке или `null`, если валидация проходит успешно.

Альтернативно вы можете использовать возможности [валидатора](/docs/{{version}}/validation) Laravel. Для этого предоставьте в аргумент `validate` массив, содержащий имя атрибута и желаемые правила проверки:

```php
$password = password(
    label: 'What is your password?',
    validate: ['password' => 'min:8']
);
```

<a name="confirm"></a>
### Подтверждение

Если вам нужно запросить у пользователя подтверждение "да или нет", вы можете использовать функцию `confirm`. Пользователи могут использовать стрелки или нажать `y` или `n`, чтобы выбрать свой ответ. Эта функция вернет либо `true`, либо `false`.

```php
use function Laravel\Prompts\confirm;

$confirmed = confirm('Do you accept the terms?');
```

Вы также можете включить значение по умолчанию, настраиваемые названия для меток "Да" и "Нет" и информационную подсказку:

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    default: false,
    yes: 'I accept',
    no: 'I decline',
    hint: 'The terms must be accepted to continue.'
);
```

<a name="confirm-required"></a>
#### Обязательное "Да"

При необходимости вы можете потребовать от ваших пользователей выбрать "Да", передав аргумент `required`:

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: true
);
```

Если вы хотите настроить сообщение об ошибке валидации, вы также можете передать строку:

```php
$confirmed = confirm(
    label: 'Do you accept the terms?',
    required: 'You must accept the terms to continue.'
);
```

<a name="select"></a>
### Выбор

Если вам нужно, чтобы пользователь выбрал из предопределенного набора вариантов, вы можете использовать функцию `select`:

```php
use function Laravel\Prompts\select;

$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner']
);
```

Вы также можете указать значение по умолчанию и информационную подсказку:

```php
$role = select(
    label: 'What role should the user have?',
    options: ['Member', 'Contributor', 'Owner'],
    default: 'Owner',
    hint: 'The role may be changed at any time.'
);
```

Вы также можете передать ассоциативный массив в аргументе `options`, чтобы вернуть выбранный ключ вместо его значения:

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    default: 'owner'
);
```

При наличии более пяти вариантов будет использоваться прокрутка списка.  Вы можете настроить это, передав аргумент scroll:

```php
$role = select(
    label: 'Which category would you like to assign?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

<a name="select-validation"></a>
#### Дополнительная валидация

В отличие от других, функция `select` не принимает аргумент `required`, потому что невозможно выбрать ничего. Однако, вы можете передать замыкание в аргумент `validate`, если вам нужно представить вариант, но предотвратить его выбор:

```php
$role = select(
    label: 'What role should the user have?',
    options: [
        'member' => 'Member',
        'contributor' => 'Contributor',
        'owner' => 'Owner',
    ],
    validate: fn (string $value) =>
        $value === 'owner' && User::where('role', 'owner')->exists()
            ? 'An owner already exists.'
            : null
);
```

Если аргумент `options` является ассоциативным массивом, то замыкание получит выбранный ключ, в противном случае оно получит выбранное значение. Замыкание может вернуть сообщение об ошибке или `null`, если валидация прошла успешно.

<a name="multiselect"></a>
### Множественный выбор

Если вам нужно, чтобы пользователь мог выбирать несколько вариантов, вы можете использовать функцию `multiselect`:

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete']
);
```

Вы также можете указать значения по умолчанию и информационную подсказку:

```php
use function Laravel\Prompts\multiselect;

$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: ['Read', 'Create', 'Update', 'Delete'],
    default: ['Read', 'Create'],
    hint: 'Permissions may be updated at any time.'
);
```

Вы также можете передать ассоциативный массив в аргумент `options`, чтобы возвращались ключи выбранных вариантов вместо их значений:

```php
$permissions = multiselect(
    label: 'What permissions should be assigned?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    default: ['read', 'create']
);
```

При наличии более пяти вариантов будет использоваться прокрутка списка.  Вы можете настроить это, передав аргумент scroll:

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    scroll: 10
);
```

<a name="multiselect-required"></a>
#### Обязательное значение

По умолчанию пользователь может выбирать ноль или более вариантов. Вы можете передать аргумент required, чтобы требовать один или более вариантов вместо этого:

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: true
);
```

Если вы хотите настроить сообщение об ошибке валидации, вы можете передать строку в аргумент `required`:

```php
$categories = multiselect(
    label: 'What categories should be assigned?',
    options: Category::pluck('name', 'id'),
    required: 'You must select at least one category'
);
```

<a name="multiselect-validation"></a>
#### Дополнительная валидация

Вы можете передать замыкание в аргумент `validate`, если вам нужно представить вариант, но предотвратить его выбор:

```php
$permissions = multiselect(
    label: 'What permissions should the user have?',
    options: [
        'read' => 'Read',
        'create' => 'Create',
        'update' => 'Update',
        'delete' => 'Delete',
    ],
    validate: fn (array $values) => ! in_array('read', $values)
        ? 'All users require the read permission.'
        : null
);
```

Если аргумент `options` является ассоциативным массивом, то замыкание получит выбранные ключи, в противном случае оно получит выбранные значения. Замыкание может вернуть сообщение об ошибке или `null`, если валидация прошла успешно.

<a name="suggest"></a>
### Подсказка

Функция `suggest` может использоваться для предоставления автозаполнения возможных вариантов. Пользователь все равно может ввести любой ответ, независимо от подсказок автозаполнения:

```php
use function Laravel\Prompts\suggest;

$name = suggest('What is your name?', ['Taylor', 'Dayle']);
```

В качестве альтернативы, вы можете передать замыкание вторым аргументом в функцию `suggest`. Замыкание будет вызываться каждый раз, когда пользователь вводит символ. Замыкание должно принимать строку, содержащую ввод пользователя до этого момента, и возвращать массив вариантов для автозаполнения:

```php
$name = suggest(
    label: 'What is your name?',
    options: fn ($value) => collect(['Taylor', 'Dayle'])
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
)
```

Вы также можете включить плейсхолдер текста, значение по умолчанию и информационную подсказку:

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    placeholder: 'E.g. Taylor',
    default: $user?->name,
    hint: 'This will be displayed on your profile.'
);
```

<a name="suggest-required"></a>
#### Обязательные значения

Если вам необходимо, чтобы было введено значение, вы можете передать аргумент `required`:

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: true
);
```

Если вы хотите настроить сообщение об ошибке валидации, вы также можете передать строку:

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    required: 'Your name is required.'
);
```

<a name="suggest-validation"></a>
#### Дополнительная валидация

Наконец, если вам нужно выполнить дополнительную логику валидации, вы можете передать замыкание в аргумент `validate`:

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

Замыкание получит введенное значение и может вернуть сообщение об ошибке или `null`, если валидация проходит успешно.

Альтернативно вы можете использовать возможности [валидатора](/docs/{{version}}/validation) Laravel. Для этого предоставьте в аргумент `validate` массив, содержащий имя атрибута и желаемые правила проверки:

```php
$name = suggest(
    label: 'What is your name?',
    options: ['Taylor', 'Dayle'],
    validate: ['name' => 'required|min:3|max:255']
);
```

<a name="search"></a>
### Поиск

Если у вас много вариантов для выбора пользователем, функция `search` позволяет пользователю вводить запрос поиска для фильтрации результатов, прежде чем использовать клавиши со стрелками для выбора параметра::

```php
use function Laravel\Prompts\search;

$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

Замыкание получит текст, введенный пользователем, и должно вернуть массив вариантов. Если вы возвращаете ассоциативный массив, то будет возвращен выбранный ключ, в противном случае будет возвращено его значение.

При фильтрации массива, в который вы собираетесь вернуть значение, вам следует использовать функцию `array_values` ​​или метод Collection `values`, чтобы гарантировать, что массив не станет ассоциативным:

```php
$names = collect(['Taylor', 'Abigail']);

$selected = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => $names
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
        ->values()
        ->all(),
);
```

Вы также можете включить плейсхолдер и информационную подсказку:

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

При наличии более пяти аргументов будет использоваться прокрутка списка. Вы можете настроить это, передав аргумент `scroll`:

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

<a name="search-validation"></a>
#### Дополнительная валидация

Если вы хотите выполнить дополнительную логику валидации, вы можете передать замыкание в аргумент `validate`:

```php
$id = search(
    label: 'Search for the user that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (int|string $value) {
        $user = User::findOrFail($value);

        if ($user->opted_out) {
            return 'This user has opted-out of receiving mail.';
        }
    }
);
```

Если замыкание `options` возвращает ассоциативный массив, то замыкание получит выбранный ключ, в противном случае оно получит выбранное значение. Замыкание может вернуть сообщение об ошибке или `null`, если валидация прошла успешно.

<a name="multisearch"></a>
### Множественный поиск

Если у вас много вариантов для поиска и вам нужно, чтобы пользователь мог выбирать несколько элементов, функция `multisearch` позволяет пользователю вводить запрос поиска для фильтрации результатов перед выбором вариантов с помощью стрелок и пробела:

```php
use function Laravel\Prompts\multisearch;

$ids = multisearch(
    'Search for the users that should receive the mail',
    fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : []
);
```

Замыкание получит текст, введенный пользователем до сих пор, и должно вернуть массив вариантов. Если вы возвращаете ассоциативный массив, то будут возвращены ключи выбранных вариантов; в противном случае будут возвращены их значения.

При фильтрации массива, в который вы собираетесь вернуть значение, вам следует использовать функцию `array_values` ​​или метод Collection `values`, чтобы гарантировать, что массив не станет ассоциативным:

```php
$names = collect(['Taylor', 'Abigail']);

$selected = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => $names
        ->filter(fn ($name) => Str::contains($name, $value, ignoreCase: true))
        ->values()
        ->all(),
);
```

Вы также можете включить плейсхолдер текста и информационную подсказку:

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    placeholder: 'E.g. Taylor Otwell',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    hint: 'The user will receive an email immediately.'
);
```

До пяти вариантов будет отображаться до начала прокрутки списка. Вы можете настроить это, указав аргумент `scroll`:

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    scroll: 10
);
```

<a name="multisearch-required"></a>
#### Обязательное значение

По умолчанию пользователь может выбрать ноль или более вариантов. Вы можете передать аргумент required, чтобы вместо этого требовать хотя бы один вариант:

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: true
);
```

Если вы хотите настроить сообщение об ошибке валидации, вы также можете передать строку в аргумент `required`:

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    required: 'You must select at least one user.'
);
```

<a name="multisearch-validation"></a>
#### Дополнительная валидация

Если вам нужно выполнить дополнительную логику валидации, вы можете передать замыкание в аргумент `validate`:

```php
$ids = multisearch(
    label: 'Search for the users that should receive the mail',
    options: fn (string $value) => strlen($value) > 0
        ? User::whereLike('name', "%{$value}%")->pluck('name', 'id')->all()
        : [],
    validate: function (array $values) {
        $optedOut = User::whereLike('name', '%a%')->findMany($values);

        if ($optedOut->isNotEmpty()) {
            return $optedOut->pluck('name')->join(', ', ', and ').' have opted out.';
        }
    }
);
```

Если замыкание `options` возвращает ассоциативный массив, то замыкание получит выбранные ключи; в противном случае оно получит выбранные значения. Замыкание может вернуть сообщение об ошибке или `null`, если валидация прошла успешно.

<a name="pause"></a>
### Пауза

Функция `pause` может использоваться для отображения информационного текста пользователю и ожидания его подтверждения продолжения нажатием клавиш Enter / Return:

```php
use function Laravel\Prompts\pause;

pause('Press ENTER to continue.');
```

<a name="transforming-input-before-validation"></a>
## Преобразование входных данных перед проверкой

Иногда вам может потребоваться преобразовать вводимые данные до того, как произойдет проверка. Например, вы можете удалить пробелы из любых предоставленных строк. Для этого многие функции приглашения предоставляют аргумент `transform`, который принимает замыкание:

```php
$name = text(
    label: 'What is your name?',
    transform: fn (string $value) => trim($value),
    validate: fn (string $value) => match (true) {
        strlen($value) < 3 => 'The name must be at least 3 characters.',
        strlen($value) > 255 => 'The name must not exceed 255 characters.',
        default => null
    }
);
```

<a name="forms"></a>
## Формы

Часто у вас будет несколько подсказок, которые будут отображаться последовательно для сбора информации перед выполнением дополнительных действий. Вы можете использовать функцию `form` для создания сгруппированного набора подсказок, которые пользователь должен выполнить:

```php
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true)
    ->password('What is your password?', validate: ['password' => 'min:8'])
    ->confirm('Do you accept the terms?')
    ->submit();
```

Метод `submit` вернет числовой индексированный массив, содержащий все ответы на запросы формы. Однако вы можете указать имя для каждого приглашения с помощью аргумента `name`. Если указано имя, доступ к ответу на указанное приглашение можно получить по этому имени:

```php
use App\Models\User;
use function Laravel\Prompts\form;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->password(
        label: 'What is your password?',
        validate: ['password' => 'min:8'],
        name: 'password'
    )
    ->confirm('Do you accept the terms?')
    ->submit();

User::create([
    'name' => $responses['name'],
    'password' => $responses['password'],
]);
```

Основным преимуществом использования функции `form` является возможность для пользователя вернуться к предыдущим запросам в форме с помощью `CTRL + U`. Это позволяет пользователю исправлять ошибки или изменять выбор без необходимости отмены и перезапуска всей формы.

Если вам нужен более детальный контроль над подсказкой в ​​форме, вы можете вызвать метод `add` вместо прямого вызова одной из функций подсказки. Методу `add` передаются все предыдущие ответы, предоставленные пользователем:

```php
use function Laravel\Prompts\form;
use function Laravel\Prompts\outro;

$responses = form()
    ->text('What is your name?', required: true, name: 'name')
    ->add(function ($responses) {
        return text("How old are you, {$responses['name']}?");
    }, name: 'age')
    ->submit();

outro("Your name is {$responses['name']} and you are {$responses['age']} years old.");
```

<a name="informational-messages"></a>
## Информационные Сообщения

Функции `note`, `info`, `warning`, `error` и `alert` могут быть использованы для отображения информационных сообщений:

```php
use function Laravel\Prompts\info;

info('Package installed successfully.');
```

<a name="tables"></a>
## Таблицы

Функция `table` упрощает отображение нескольких строк и столбцов данных. Все, что вам нужно сделать, это указать имена столбцов и данные для таблицы:

```php
use function Laravel\Prompts\table;

table(
    headers: ['Name', 'Email'],
    rows: User::all(['name', 'email'])->toArray()
);
```

<a name="spin"></a>
## Spin

Функция `spin` отображает спиннер вместе с необязательным сообщением во время выполнения указанного обратного вызова. Она служит для обозначения выполнения процессов и возвращает результаты обратного вызова по его завершении:

```php
use function Laravel\Prompts\spin;

$response = spin(
    message: 'Fetching response...',
    callback: fn () => Http::get('http://example.com')
);
```

> [!WARNING]
> Функция `spin` требует наличие расширения PHP `pcntl` для анимации спиннера. Когда это расширение недоступно, вместо этого будет отображаться статическая версия спиннера.

<a name="progress"></a>
## Прогресс-бар

Для длительных задач может быть полезно показать полосу прогресса, которая информирует пользователей о том, насколько завершена задача. Используя функцию `progress`, Laravel будет отображать полосу прогресса и продвигать ее для каждой итерации по заданному итерируемому значению:

```php
use function Laravel\Prompts\progress;

$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: fn ($user) => $this->performTask($user)
);
```

Функция `progress` действует как функция map и вернет массив, содержащий возвращаемое значение каждой итерации вашего обратного вызова.

Обратный вызов также может принимать экземпляр `Laravel\Prompts\Progress`, что позволяет вам изменять метку и подсказку на каждой итерации:

```php
$users = progress(
    label: 'Updating users',
    steps: User::all(),
    callback: function ($user, $progress) {
        $progress
            ->label("Updating {$user->name}")
            ->hint("Created on {$user->created_at}");

        return $this->performTask($user);
    },
    hint: 'This may take some time.'
);
```

Иногда вам может потребоваться больше ручного контроля над тем, как продвигается полоса прогресса. Сначала определите общее количество шагов, через которые будет проходить процесс. Затем продвигайте полосу прогресса с помощью метода `advance` после обработки каждого элемента:

```php
$progress = progress(label: 'Updating users', steps: 10);

$users = User::all();

$progress->start();

foreach ($users as $user) {
    $this->performTask($user);

    $progress->advance();
}

$progress->finish();
```

<a name="clear"></a>
## Очистка терминала

Функция `clear` может использоваться для очистки пользовательского терминала:

```
use function Laravel\Prompts\clear;

clear();
```

<a name="terminal-considerations"></a>
## Учет Особенностей Терминала

<a name="terminal-width"></a>
#### Ширина Терминала

Если длина какой-либо метки, варианта или сообщения о валидации превышает количество "столбцов" в терминале пользователя, она будет автоматически усечена до соответствия. Рассмотрите возможность минимизации длины этих строк, если ваши пользователи могут использовать более узкие терминалы. Обычно безопасная максимальная длина составляет 74 символа для поддержки терминала шириной 80 символов.

<a name="terminal-height"></a>
#### Высота Терминала

Для всех запросов, которые принимают аргумент `scroll`, настроенное значение будет автоматически уменьшено для соответствия высоте терминала пользователя, включая место для сообщения о валидации.

<a name="fallbacks"></a>
## Неподдерживаемые Окружения и Резервные Варианты

Laravel Prompts поддерживает macOS, Linux и Windows с использованием WSL. Из-за ограничений в версии PHP для Windows в настоящее время невозможно использовать Laravel Prompts на Windows вне WSL.

По этой причине Laravel Prompts поддерживает откат к альтернативной реализации, такой как [Symfony Console Question Helper](https://symfony.com/doc/7.0/components/console/helpers/questionhelper.html).

> [!NOTE]
> При использовании Laravel Prompts с фреймворком Laravel резервные варианты для каждого запроса настроены для вас и будут автоматически включены в неподдерживаемых окружениях.

<a name="fallback-conditions"></a>
#### Условия Резервных Вариантов

Если вы не используете Laravel или нуждаетесь в настройке условий использования резервного поведения, вы можете передать булево значение методу `fallbackWhen` статического класса `Prompt`:

```php
use Laravel\Prompts\Prompt;

Prompt::fallbackWhen(
    ! $input->isInteractive() || windows_os() || app()->runningUnitTests()
);
```

<a name="fallback-behavior"></a>
#### Поведение Резервного Варианта

Если вы не используете Laravel или вам нужно настроить поведение резервного варианта, вы можете передать замыкание методу `fallbackUsing` статического класса каждого prompt:

```php
use Laravel\Prompts\TextPrompt;
use Symfony\Component\Console\Question\Question;
use Symfony\Component\Console\Style\SymfonyStyle;

TextPrompt::fallbackUsing(function (TextPrompt $prompt) use ($input, $output) {
    $question = (new Question($prompt->label, $prompt->default ?: null))
        ->setValidator(function ($answer) use ($prompt) {
            if ($prompt->required && $answer === null) {
                throw new \RuntimeException(
                    is_string($prompt->required) ? $prompt->required : 'Required.'
                );
            }

            if ($prompt->validate) {
                $error = ($prompt->validate)($answer ?? '');

                if ($error) {
                    throw new \RuntimeException($error);
                }
            }

            return $answer;
        });

    return (new SymfonyStyle($input, $output))
        ->askQuestion($question);
});
```

Резервные варианты должны быть настроены индивидуально для каждого класса prompt. Замыкание будет получать экземпляр класса prompt и должно возвращать соответствующий тип для prompt.
