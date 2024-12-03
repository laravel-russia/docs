---
git: 8758bd2c244370939f1246f1d32138ec478c01e1
---

# Precognition

<a name="introduction"></a>
## Введение

Laravel Precognition позволяет предвидеть результат будущего HTTP-запроса. Одним из основных случаев использования Precognition является возможность обеспечить "живую" валидацию для вашего фронтенда на JavaScript. Precognition особенно хорошо работает с [стартовыми наборами](/docs/{{version}}/starter-kits) Laravel, основанными на Inertia.

Когда Laravel получает "предвиденный запрос", он выполнит всю промежуточную обработку маршрута и разрешит зависимости контроллера маршрута, включая валидацию [запросов формы](/docs/{{version}}/validation#form-request-validation) - но он фактически не выполнит метод контроллера маршрута.

<a name="live-validation"></a>
## Живая валидация

<a name="using-vue"></a>
### Использование Vue

С использованием Laravel Precognition вы можете предоставить пользователям опыт живой валидации без необходимости дублирования правил валидации в вашем фронтенд-приложении Vue. Чтобы проиллюстрировать, как это работает, давайте сделаем форму для создания новых пользователей в нашем приложении.

Сначала, чтобы включить Precognition для маршрута, к нему должно быть добавлено middleware `HandlePrecognitiveRequests`. Вы также должны создать [запрос формы](/docs/{{version}}/validation#form-request-validation), чтобы разместить правила валидации маршрута:

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

Далее, вам следует установить через NPM помощники фронтенда Laravel Precognition для Vue :

```shell
npm install laravel-precognition-vue
```

После установки пакета Laravel Precognition вы можете создать объект формы, используя функцию `useForm` Precognition, указав метод HTTP (`post`), целевой URL (`/users`) и начальные данные формы.

Затем, чтобы включить живую валидацию, вызовите метод `validate` формы для каждого события `change` поля ввода, предоставив имя поля:

```vue
<script setup>
import { useForm } from 'laravel-precognition-vue';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = () => form.submit();
</script>

<template>
    <form @submit.prevent="submit">
        <label for="name">Name</label>
        <input
            id="name"
            v-model="form.name"
            @change="form.validate('name')"
        />
        <div v-if="form.invalid('name')">
            {{ form.errors.name }}
        </div>

        <label for="email">Email</label>
        <input
            id="email"
            type="email"
            v-model="form.email"
            @change="form.validate('email')"
        />
        <div v-if="form.invalid('email')">
            {{ form.errors.email }}
        </div>

        <button :disabled="form.processing">
            Create User
        </button>
    </form>
</template>
```

Теперь, когда пользователь заполняет форму, Precognition будет предоставлять вывод живой валидации на основе правил валидации в запросе формы маршрута. Когда изменяются входные данные формы, будет отправлен запрос на "предвидение" валидации с задержкой времени, определенной функцией setValidationTimeout формы в Laravel.

```js
form.setValidationTimeout(3000);
```

Когда запрос на валидацию находится в процессе выполнения, свойство `validating` формы будет равно `true`:

```html
<div v-if="form.validating">
    Validating...
</div>
```

Любые ошибки валидации, возвращенные во время запроса на валидацию или отправки формы, автоматически добавятся в объект `errors` формы:

```html
<div v-if="form.invalid('email')">
    {{ form.errors.email }}
</div>
```

Вы можете определить, есть ли в форме какие-либо ошибки, используя свойство `hasErrors` формы:

```html
<div v-if="form.hasErrors">
    <!-- ... -->
</div>
```

Вы также можете определить, прошло ли валидацию введённое значение или нет, передав имя поля ввода функциям `valid` и `invalid` формы соответственно:

```html
<span v-if="form.valid('email')">
    ✅
</span>

<span v-else-if="form.invalid('email')">
    ❌
</span>
```

> [!WARNING]
> Поле ввода будет считаться валидным или невалидным только после его изменения и получения ответа о валидации.

Если вы проверяете подмножество вводимых данных формы с помощью Precognition, может быть полезно вручную очистить ошибки. Для этого можно использовать функцию `forgetError` формы:

```html
<input
    id="avatar"
    type="file"
    @change="(e) => {
        form.avatar = e.target.files[0]

        form.forgetError('avatar')
    }"
>
```

Как мы видели, вы можете подключиться к событию ввода `change` и проверять отдельные входные данные по мере взаимодействия с ними пользователя; однако вам может потребоваться проверить входные данные, с которыми пользователь еще не взаимодействовал. Это часто встречается при создании «мастера», когда вы хотите проверить все видимые входные данные, независимо от того, взаимодействовал ли с ними пользователь или нет, прежде чем перейти к следующему шагу.

Чтобы сделать это с помощью Precognition, вам следует пометить поля, которые вы хотите проверить, как «затронутые», передав их имена методу `touch`. Затем вызовите метод `validate` с обратными вызовами `onSuccess` или `onValidationError`:

```html
<button
    type="button"
    @click="form.touch(['name', 'email', 'phone']).validate({
        onSuccess: (response) => nextStep(),
        onValidationError: (response) => /* ... */,
    })"
>Next Step</button>
```

Конечно, вы также можете выполнить код при получении ответа на отправку формы. Функция `submit` формы возвращает обещание(promise) запроса Axios. Это обеспечивает удобный способ доступа к данным ответа, сброса входных данных формы при успешной отправке или обработки неудачного запроса:

```js
const submit = () => form.submit()
    .then(response => {
        form.reset();

        alert('User created.');
    })
    .catch(error => {
        alert('An error occurred.');
    });
```

Вы можете определить, выполняется ли запрос на отправку формы, проверив свойство `processing` формы:

```html
<button :disabled="form.processing">
    Submit
</button>
```

<a name="using-vue-and-inertia"></a>
### Использование Vue и Inertia

> [!NOTE]
> Если вы хотите быстро начать разработку вашего приложения Laravel с использованием Vue и Inertia, рассмотрите возможность использования одного из наших [стартовых наборов](/docs/{{version}}/starter-kits). Стартовые наборы Laravel предоставляют основу для аутентификации на стороне сервера и на стороне клиента для вашего нового приложения Laravel.

Перед использованием Precognition с Vue и Inertia, убедитесь, что вы ознакомились с нашей общей документацией по [использованию Precognition с Vue](#using-vue). При использовании Vue с Inertia вам потребуется установить совместимую с Inertia библиотеку Precognition с помощью NPM:

```shell
npm install laravel-precognition-vue-inertia
```

После установки функция `useForm` от Precognition будет возвращать помощник [формы](https://inertiajs.com/forms#form-helper) Inertia, дополненный функциями валидации,  о которых говорилось выше.

Метод `submit` помощника формы был оптимизирован, и теперь не требуется указывать метод HTTP или URL. Вместо этого вы можете передать [параметры посещения](https://inertiajs.com/manual-visits) Inertia в качестве единственного аргумента. Кроме того, метод `submit` не возвращает Promise, как в приведенном выше примере с Vue. Вместо этого вы можете предоставить любой из поддерживаемых Inertia [обратных вызовов событий](https://inertiajs.com/manual-visits#event-callbacks) в параметрах посещения, переданных методу `submit`:

```vue
<script setup>
import { useForm } from 'laravel-precognition-vue-inertia';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = () => form.submit({
    preserveScroll: true,
    onSuccess: () => form.reset(),
});
</script>
```

<a name="using-react"></a>
### Использование React

С помощью Laravel Precognition вы можете предложить пользователям живую валидацию возможности без необходимости дублирования правил валидации в вашем React фронтенде. Чтобы проиллюстрировать, как это работает, давайте сделаем форму для создания новых пользователей в нашем приложении.

Во-первых, чтобы включить Precognition для маршрута, необходимо добавить middleware `HandlePrecognitiveRequests` в определение маршрута. Также следует создать [запрос формы](/docs/{{version}}/validation#form-request-validation), чтобы разместить правила валидации маршрута:

```php
use App\Http\Requests\StoreUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (StoreUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

Затем вы должны установить помощники фронтенда Laravel Precognition для React с помощью NPM:

```shell
npm install laravel-precognition-react
```

После установки пакета Laravel Precognition вы можете создать объект формы, используя функцию `useForm` Precognition, указав метод HTTP (`post`), целевой URL (`/users`) и начальные данные формы.

Чтобы включить живую валидацию, вы должны прослушивать события `change` и `blur` для каждого поля ввода. В обработчике события `change` вы должны установить данные формы с помощью функции `setData`, передав имя поля ввода и новое значение. Затем, в обработчике события `blur`, вызовите метод `validate` формы, указав имя поля:

```jsx
import { useForm } from 'laravel-precognition-react';

export default function Form() {
    const form = useForm('post', '/users', {
        name: '',
        email: '',
    });

    const submit = (e) => {
        e.preventDefault();

        form.submit();
    };

    return (
        <form onSubmit={submit}>
            <label htmlFor="name">Name</label>
            <input
                id="name"
                value={form.data.name}
                onChange={(e) => form.setData('name', e.target.value)}
                onBlur={() => form.validate('name')}
            />
            {form.invalid('name') && <div>{form.errors.name}</div>}

            <label htmlFor="email">Email</label>
            <input
                id="email"
                value={form.data.email}
                onChange={(e) => form.setData('email', e.target.value)}
                onBlur={() => form.validate('email')}
            />
            {form.invalid('email') && <div>{form.errors.email}</div>}

            <button disabled={form.processing}>
                Create User
            </button>
        </form>
    );
};
```

Теперь, по мере заполнения формы пользователем, Precognition будет предоставлять живой вывод валидации на основе правил валидации в запросе формы маршрута. При изменении содержимого полей ввода  формы, будет отправлен запрос с задержкой на "предвидение" валидации в ваше приложение Laravel. Вы можете настроить время задержки, вызвав функцию `setValidationTimeout` формы:

```js
form.setValidationTimeout(3000);
```

Когда запрос на валидацию находится в процессе выполнения, свойство `validating` формы будет равно `true`:

```jsx
{form.validating && <div>Validating...</div>}
```

Любые ошибки валидации, возвращенные во время запроса на валидацию или отправки формы, автоматически заполнят объект `errors` формы:

```jsx
{form.invalid('email') && <div>{form.errors.email}</div>}
```

Вы можете определить, есть ли в форме какие-либо ошибки, используя свойство `hasErrors` формы:

```jsx
{form.hasErrors && <div><!-- ... --></div>}
```

Вы также можете определить, прошло ли валидацию введённое значение или нет, передав имя поля ввода функциям `valid` и `invalid` формы соответственно:

```jsx
{form.valid('email') && <span>✅</span>}

{form.invalid('email') && <span>❌</span>}
```

> [!WARNING]
> Поле ввода будет считаться валидным или невалидным только после его изменения и получения ответа о валидации.

Если вы проверяете подмножество вводимых данных формы с помощью Precognition, может быть полезно вручную очистить ошибки. Для этого можно использовать функцию `forgetError` формы:

```jsx
<input
    id="avatar"
    type="file"
    onChange={(e) => {
        form.setData('avatar', e.target.value);

        form.forgetError('avatar');
    }}
>
```

Как мы видели, вы можете подключиться к событию ввода `blur` и проверять отдельные вводимые данные по мере взаимодействия с ними пользователя; однако вам может потребоваться проверить входные данные, с которыми пользователь еще не взаимодействовал. Это часто встречается при создании «мастера», где вы хотите проверить все видимые входные данные, независимо от того, взаимодействовал ли с ними пользователь или нет, прежде чем перейти к следующему шагу.

Чтобы сделать это с помощью Precognition, вам следует пометить поля, которые вы хотите проверить, как «затронутые», передав их имена методу `touch`. Затем вызовите метод `validate` с обратными вызовами `onSuccess` или `onValidationError`:

```jsx
<button
    type="button"
    onClick={() => form.touch(['name', 'email', 'phone']).validate({
        onSuccess: (response) => nextStep(),
        onValidationError: (response) => /* ... */,
    })}
>Next Step</button>
```

Конечно, вы также можете выполнять код после ответа на отправку формы. Функция `submit` формы возвращает promise запроса Axios. Это обеспечивает удобный способ доступа к данным ответа, сброса входных данных формы при успешной отправке или обработки неудачного запроса:

```js
const submit = (e) => {
    e.preventDefault();

    form.submit()
        .then(response => {
            form.reset();

            alert('User created.');
        })
        .catch(error => {
            alert('An error occurred.');
        });
};
```

Вы можете определить, выполняется ли запрос на отправку формы, проверив свойство `processing` формы:

```html
<button disabled={form.processing}>
    Submit
</button>
```

<a name="using-react-and-inertia"></a>
### Использование React и Inertia

> [!NOTE]
> Если вы хотите быстро начать разработку вашего приложения Laravel с использованием React и Inertia, рассмотрите возможность использования одного из наших [стартовых наборов](/docs/{{version}}/starter-kits). Стартовые наборы Laravel предоставляют основу для аутентификации на стороне сервера и на стороне клиента для вашего нового приложения Laravel.

Перед использованием Precognition с React и Inertia убедитесь, что вы ознакомились с нашей общей документацией по [использованию Precognition с React](#using-react). При использовании React с Inertia вам потребуется установить совместимую с Inertia библиотеку Precognition с помощью NPM:

```shell
npm install laravel-precognition-react-inertia
```

После установки Precognition функция `useForm` будет возвращать помощника формы Inertia, дополненного функциями валидации, описанными выше.

Метод `submit` помощника формы был оптимизирован, и теперь не требуется указывать метод HTTP или URL. Вместо этого вы можете передать параметры [посещения](https://inertiajs.com/manual-visits) Inertia в качестве единственного аргумента. Кроме того, метод `submit` не возвращает Promise, как в приведенном выше примере с React. Вместо этого вы можете предоставить любой из поддерживаемых Inertia [обратных вызывов событий](https://inertiajs.com/manual-visits#event-callbacks) в параметрах посещения, переданных методу `submit`:

```js
import { useForm } from 'laravel-precognition-react-inertia';

const form = useForm('post', '/users', {
    name: '',
    email: '',
});

const submit = (e) => {
    e.preventDefault();

    form.submit({
        preserveScroll: true,
        onSuccess: () => form.reset(),
    });
};
```

<a name="using-alpine"></a>
### Использование Alpine и Blade

С помощью Laravel Precognition вы можете предоставить вашим пользователям возможность живой валидации, не дублируя правила валидации в вашем Alpine фронтенде. Чтобы проиллюстрировать, как это работает, давайте сделаем форму для создания новых пользователей в нашем приложении.

Сначала, чтобы включить Precognition для маршрута, к его определению следует добавить middleware `HandlePrecognitiveRequests`. Также вы должны создать [запрос формы](/docs/{{version}}/validation#form-request-validation), чтобы разместить правила валидации маршрута:

```php
use App\Http\Requests\CreateUserRequest;
use Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests;

Route::post('/users', function (CreateUserRequest $request) {
    // ...
})->middleware([HandlePrecognitiveRequests::class]);
```

Затем вы должны установить помощники фронтенда Laravel Precognition для Alpine с помощью NPM:

```shell
npm install laravel-precognition-alpine
```

Затем зарегистрируйте плагин Precognition с Alpine в вашем файле `resources/js/app.js`:

```js
import Alpine from 'alpinejs';
import Precognition from 'laravel-precognition-alpine';

window.Alpine = Alpine;

Alpine.plugin(Precognition);
Alpine.start();
```

С установленным и зарегистрированным пакетом Laravel Precognition теперь вы можете создать объект формы, используя "магию" `$form` Precognition, указав метод HTTP (`post`), целевой URL (`/users`) и начальные данные формы.

Для включения живой валидации вы должны привязать данные формы к соответствующему полю ввода, а затем прослушивать событие `change` для каждого поля ввода. В обработчике события `change` вы должны вызвать метод `validate` формы, указав имя поля ввода:

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '',
        email: '',
    }),
}">
    @csrf
    <label for="name">Name</label>
    <input
        id="name"
        name="name"
        x-model="form.name"
        @change="form.validate('name')"
    />
    <template x-if="form.invalid('name')">
        <div x-text="form.errors.name"></div>
    </template>

    <label for="email">Email</label>
    <input
        id="email"
        name="email"
        x-model="form.email"
        @change="form.validate('email')"
    />
    <template x-if="form.invalid('email')">
        <div x-text="form.errors.email"></div>
    </template>

    <button :disabled="form.processing">
        Create User
    </button>
</form>
```

Теперь, когда пользователь заполняет форму, Precognition будет предоставлять живой вывод валидации на основе правил валидации в запросе формы маршрута. При изменении содержимого полей ввода формы будет отправлен запрос на "предвидение" валидации с задержкой в ваше приложение Laravel. Вы можете настроить время задержки, вызвав функцию `setValidationTimeout` формы:

```js
form.setValidationTimeout(3000);
```

Когда запрос на валидацию находится в процессе выполнения, свойство `validating` формы будет равно `true`:

```html
<template x-if="form.validating">
    <div>Validating...</div>
</template>
```

Любые ошибки валидации, возвращенные во время запроса на валидацию или отправки формы, автоматически заполнят объект `errors` формы:

```html
<template x-if="form.invalid('email')">
    <div x-text="form.errors.email"></div>
</template>
```

Вы можете определить, есть ли у формы какие-либо ошибки, используя свойство `hasErrors` формы:

```html
<template x-if="form.hasErrors">
    <div><!-- ... --></div>
</template>
```

Вы также можете определить, прошло ли введённое значение валидацию или нет, передав имя поля ввода функциям `valid` и `invalid` формы соответственно:

```html
<template x-if="form.valid('email')">
    <span>✅</span>
</template>

<template x-if="form.invalid('email')">
    <span>❌</span>
</template>
```

> [!WARNING]
> Входное значение формы будет отображаться как валидное или валидное только после того, как оно изменится, и будет получен ответ на валидацию.

Как мы видели, вы можете подключиться к событию ввода `change` и проверять отдельные входные данные по мере взаимодействия с ними пользователя; однако вам может потребоваться проверить входные данные, с которыми пользователь еще не взаимодействовал. Это часто встречается при создании «мастера», когда вы хотите проверить все видимые входные данные, независимо от того, взаимодействовал ли с ними пользователь или нет, прежде чем перейти к следующему шагу.

Чтобы сделать это с помощью Precognition, вам следует пометить поля, которые вы хотите проверить, как «затронутые», передав их имена методу `touch`. Затем вызовите метод `validate` с обратными вызовами `onSuccess` или `onValidationError`:

```html
<button
    type="button"
    @change="form.touch(['name', 'email', 'phone']).validate({
        onSuccess: (response) => nextStep(),
        onValidationError: (response) => /* ... */,
    })"
>Next Step</button>
```

Вы можете определить, выполняется ли запрос на отправку формы, проверив свойство `processing` формы:

```html
<button :disabled="form.processing">
    Submit
</button>
```

<a name="repopulating-old-form-data"></a>
#### Восстановление старых данных формы

В приведенном выше примере создания пользователя мы используем Precognition для выполнения живой валидации; однако мы выполняем традиционную отправку формы на сервер для её обработки. Поэтому форма должна быть заполнена любыми "старыми" данными ввода и ошибками валидации, возвращенными после отправки формы на сервер:

```html
<form x-data="{
    form: $form('post', '/register', {
        name: '{{ old('name') }}',
        email: '{{ old('email') }}',
    }).setErrors({{ Js::from($errors->messages()) }}),
}">
```

В качестве альтернативы, если вы хотите отправить форму через XHR, вы можете использовать функцию `submit` формы, которая возвращает promise запроса Axios:

```html
<form
    x-data="{
        form: $form('post', '/register', {
            name: '',
            email: '',
        }),
        submit() {
            this.form.submit()
                .then(response => {
                    form.reset();

                    alert('User created.')
                })
                .catch(error => {
                    alert('An error occurred.');
                });
        },
    }"
    @submit.prevent="submit"
>
```

<a name="configuring-axios"></a>
### Настройка Axios

Библиотеки валидации Precognition используют клиент HTTP [Axios](https://github.com/axios/axios) для отправки запросов на бэкенд вашего приложения. Для удобства экземпляр Axios может быть настроен, если это необходимо для вашего приложения. Например, при использовании библиотеки `laravel-precognition-vue` вы можете добавить дополнительные заголовки запроса к каждому исходящему запросу в файле `resources/js/app.js` вашего приложения:

```js
import { client } from 'laravel-precognition-vue';

client.axios().defaults.headers.common['Authorization'] = authToken;
```

Или, если у вас уже есть настроенный экземпляр Axios для вашего приложения, вы можете указать Precognition использовать этот экземпляр вместо создания нового:

```js
import Axios from 'axios';
import { client } from 'laravel-precognition-vue';

window.axios = Axios.create()
window.axios.defaults.headers.common['Authorization'] = authToken;

client.use(window.axios)
```

> [!WARNING]
> Библиотеки Precognition, адаптированные под Inertia, будут использовать настроенный экземпляр Axios только для запросов валидации. Отправка форм всегда будет осуществляться через Inertia.

<a name="customizing-validation-rules"></a>
## Настройка правил валидации

Можно настроить правила валидации, выполняемые во время предвиденного запроса, используя метод `isPrecognitive` запроса.

Например, в форме создания пользователя мы можем проверить, что пароль не скомпрометирован только при окончательной отправке формы. Для предвиденных запросов валидации мы просто проверим, что пароль обязателен и имеет минимальную длину в 8 символов. Используя метод `isPrecognitive`, мы можем настроить правила, определенные в нашем запросе формы:

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rules\Password;

class StoreUserRequest extends FormRequest
{
    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    protected function rules()
    {
        return [
            'password' => [
                'required',
                $this->isPrecognitive()
                    ? Password::min(8)
                    : Password::min(8)->uncompromised(),
            ],
            // ...
        ];
    }
}
```

<a name="handling-file-uploads"></a>
## Обработка загрузки файлов

По умолчанию Laravel Precognition не загружает и не проверяет файлы во время предвиденного запроса валидации. Это гарантирует, что большие файлы не будут раз загружены несколько раз.

Из-за этого поведения вы должны убедиться, что в вашем приложении [настроены соответствующие правила валидации запроса формы](#customizing-validation-rules), чтобы указать, что поле обязательно только для полной отправки формы:

```php
/**
 * Get the validation rules that apply to the request.
 *
 * @return array
 */
protected function rules()
{
    return [
        'avatar' => [
            ...$this->isPrecognitive() ? [] : ['required'],
            'image',
            'mimes:jpg,png',
            'dimensions:ratio=3/2',
        ],
        // ...
    ];
}
```

Если вы хотите включить файлы в каждый запрос на валидацию, вы можете вызвать функцию `validateFiles` на вашем экземпляре формы на стороне клиента:

```js
form.validateFiles();
```

<a name="managing-side-effects"></a>
## Управление побочными эффектами

При добавлении middleware `HandlePrecognitiveRequests` к маршруту вы должны проверить, есть ли какие-либо побочные эффекты в _других_ middleware, которые следует пропустить во время предвиденного запроса.

Например, у вас может быть middleware, который увеличивает общее количество "взаимодействий" каждого пользователя с вашим приложением, но вы не хотите, чтобы предвиденные запросы учитывались как взаимодействие. Чтобы добиться этого, мы можем проверить метод `isPrecognitive` запроса перед увеличением счетчика взаимодействий:

```php
<?php

namespace App\Http\Middleware;

use App\Facades\Interaction;
use Closure;
use Illuminate\Http\Request;

class InteractionMiddleware
{
    /**
     * Handle an incoming request.
     */
    public function handle(Request $request, Closure $next): mixed
    {
        if (! $request->isPrecognitive()) {
            Interaction::incrementFor($request->user());
        }

        return $next($request);
    }
}
```

<a name="testing"></a>
## Тестирование

Если вы хотите выполнять предвиденные запросы в ваших тестах, в `TestCase` Laravel есть вспомогательная функция `withPrecognition`, которая добавит заголовок запроса `Precognition`.

Кроме того, если вы хотите проверить, что предвиденный запрос был успешным, например, не возвращал никаких ошибок валидации, вы можете использовать метод `assertSuccessfulPrecognition` на ответе:

```php tab=Pest
it('validates registration form with precognition', function () {
    $response = $this->withPrecognition()
        ->post('/register', [
            'name' => 'Taylor Otwell',
        ]);

    $response->assertSuccessfulPrecognition();

    expect(User::count())->toBe(0);
});
```

```php tab=PHPUnit
public function test_it_validates_registration_form_with_precognition()
{
    $response = $this->withPrecognition()
        ->post('/register', [
            'name' => 'Taylor Otwell',
        ]);

    $response->assertSuccessfulPrecognition();
    $this->assertSame(0, User::count());
}
```
