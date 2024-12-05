---
git: 696cad17f80605f1cf2ff68c97f60873500c55ca
---

# Laravel Cashier (Paddle)

<a name="introduction"></a>
## Введение

> [!WARNING]
> Эта документация предназначена для интеграции Cashier Paddle 2.x с Paddle Billing. Если вы все еще используете Paddle Classic, вам следует использовать [Cashier Paddle 1.x](https://github.com/laravel/cashier-paddle/tree/1.x).

[Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle) предоставляет выразительный и удобный интерфейс для служб выставления счетов по подписке [Paddle](https://paddle.com). Он обрабатывает почти все стандартные коды выставления счетов за подписку, которых вы боитесь. Помимо базового управления подписками, Cashier может обрабатывать: замену подписок, «количество» подписки, приостановку подписки, льготные периоды отмены и многое другое.

Прежде чем углубляться в Cashier Paddle, мы рекомендуем вам также просмотреть [концептуальные руководства](https://developer.paddle.com/concepts/overview) и [документацию API](https://developer.paddle.com/api-reference/overview).

<a name="upgrading-cashier"></a>
## Обновление Cashier

При обновлении до новой версии Cashier важно внимательно просмотреть [руководство по обновлению](https://github.com/laravel/cashier-paddle/blob/master/UPGRADE.md).

<a name="installation"></a>
## Установка

Сначала установите пакет Cashier для Paddle с помощью менеджера пакетов Composer:

```shell
composer require laravel/cashier-paddle
```

Далее вам следует опубликовать файлы миграции Cashier с помощью Artisan-команды `vendor:publish`:

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

Затем вам следует запустить миграцию базы данных вашего приложения. Миграция Cashier создаст новую таблицу `customers`. Кроме того, будут созданы новые таблицы `subscriptions` и `subscription_items` для хранения всех подписок ваших клиентов. Наконец, будет создана новая таблица `transactions` для хранения всех транзакций Paddle, связанных с вашими клиентами:

```shell
php artisan migrate
```

> [!WARNING]
> Чтобы убедиться, что Cashier правильно обрабатывает все события Paddle, не забудьте [настроить обработку веб-перехватчика Cashier](#handling-paddle-webhooks).

<a name="paddle-sandbox"></a>
### Paddle Sandbox

Во время локальной и промежуточной разработки вам следует [зарегистрировать учетную запись Paddle Sandbox](https://sandbox-login.paddle.com/signup). Эта учетная запись предоставит вам изолированную среду для тестирования и разработки ваших приложений без внесения реальных платежей. Вы можете использовать [номера тестовых карт Paddle](https://developer.paddle.com/concepts/pay-methods/credit-debit-card) для моделирования различных сценариев оплаты.

При использовании среды Paddle Sandbox вам следует установить для переменной среды `PADDLE_SANDBOX` значение `true` в файле `.env` вашего приложения:

```ini
PADDLE_SANDBOX=true
```

После завершения разработки приложения вы можете [подать заявку на получение учетной записи поставщика Paddle](https://paddle.com). Прежде чем ваше приложение будет запущено в производство, Paddle необходимо будет утвердить домен вашего приложения.

<a name="configuration"></a>
## Конфигурация

<a name="billable-model"></a>
### Оплачиваемая модель

Прежде чем использовать Cashier, вы должны добавить трейт `Billable` в определение вашей пользовательской модели. Эта особенность предоставляет различные методы, позволяющие выполнять общие задачи по выставлению счетов, такие как создание подписок и обновление информации о способе оплаты:

    use Laravel\Paddle\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

Если у вас есть оплачиваемые сущности, которые не являются пользователями, вы также можете добавить признак к этим классам:

    use Illuminate\Database\Eloquent\Model;
    use Laravel\Paddle\Billable;

    class Team extends Model
    {
        use Billable;
    }

<a name="api-keys"></a>
### Ключи API

Далее вам следует настроить ключи Paddle в файле `.env` вашего приложения. Вы можете получить ключи API Paddle из панели управления Paddle:

```ini
PADDLE_CLIENT_SIDE_TOKEN=your-paddle-client-side-token
PADDLE_API_KEY=your-paddle-api-key
PADDLE_RETAIN_KEY=your-paddle-retain-key
PADDLE_WEBHOOK_SECRET="your-paddle-webhook-secret"
PADDLE_SANDBOX=true
```

Переменная среды `PADDLE_SANDBOX` должна иметь значение `true`, когда вы используете [среду Paddle Sandbox](#paddle-sandbox). Переменная `PADDLE_SANDBOX` должна иметь значение `false`, если вы развертываете свое приложение в рабочей среде и используете рабочую среду Paddle.

`PADDLE_RETAIN_KEY` является необязательным и его следует устанавливать только в том случае, если вы используете Paddle с [Retain](https://developer.paddle.com/paddlejs/retain).

<a name="paddle-js"></a>
### Paddle JS

Paddle использует собственную библиотеку JavaScript для запуска виджета оформления заказа Paddle. Вы можете загрузить библиотеку JavaScript, поместив директиву Blade `@paddleJS` прямо перед закрывающим тегом `</head>` макета вашего приложения:

```blade
<head>
    ...

    @paddleJS
</head>
```

<a name="currency-configuration"></a>
### Конфигурация валюты

Вы можете указать языковой стандарт, который будет использоваться при форматировании денежных значений для отображения в счетах. Внутри Cashier использует [класс PHP NumberFormatter](https://www.php.net/manual/ru/class.numberformatter.php) для установки языкового стандарта валюты:

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> Чтобы использовать локали, отличные от `en`, убедитесь, что на вашем сервере установлено и настроено расширение PHP `ext-intl`.

<a name="overriding-default-models"></a>
### Переопределение моделей по умолчанию

Вы можете расширить модели, используемые внутри Cashier, определив свою собственную модель и расширив соответствующую модель Cashier:

    use Laravel\Paddle\Subscription as CashierSubscription;

    class Subscription extends CashierSubscription
    {
        // ...
    }

После определения вашей модели вы можете поручить Cashier использовать вашу пользовательскую модель через класс `Laravel\Paddle\Cashier`. Обычно вы должны сообщить Cashier о ваших пользовательских моделях в методе `boot` класса `App\Providers\AppServiceProvider` вашего приложения:

    use App\Models\Cashier\Subscription;
    use App\Models\Cashier\Transaction;

    /**
     * Запуск любых служб приложений.
     */
    public function boot(): void
    {
        Cashier::useSubscriptionModel(Subscription::class);
        Cashier::useTransactionModel(Transaction::class);
    }

<a name="quickstart"></a>
## Быстрый старт

<a name="quickstart-selling-products"></a>
### Продажа продуктов

> [!NOTE]
> Прежде чем использовать Paddle Checkout, вам следует определить продукты с фиксированными ценами на панели управления Paddle. Кроме того, вам следует [настроить обработку веб-перехватчиков Paddle](#handling-paddle-webhooks).

Предложение выставления счетов за продукты и подписки через ваше приложение может напугать. Однако благодаря Cashier и [Paddle's Checkout Overlay](https://www.paddle.com/billing/checkout) вы можете легко создать современную и надежную платежную интеграцию.

Чтобы взимать с клиентов плату за единовременные продукты с единовременной оплатой, мы будем использовать Cashier с Paddle Checkout Overlay для оплаты, где они предоставят свои платежные реквизиты и подтвердят свою покупку. После того, как платеж будет произведен через Checkout Overlay, клиент будет перенаправлен на выбранный вами успешный URL-адрес в вашем приложении:

    use Illuminate\Http\Request;

    Route::get('/buy', function (Request $request) {
        $checkout = $request->user()->checkout('pri_deluxe_album')
            ->returnTo(route('dashboard'));

        return view('buy', ['checkout' => $checkout]);
    })->name('checkout');

Как вы можете видеть в приведенном выше примере, мы будем использовать предоставленный Cashier метод `checkout` для создания объекта оформления заказа, чтобы предоставить покупателю счет Paddle Checkout Overlay для данного «идентификатора цены». При использовании Paddle «цены» относятся к [определенным ценам для конкретных продуктов](https://developer.paddle.com/build/products/create-products-prices).

При необходимости метод `checkout` автоматически создаст клиента в Paddle и соединит эту запись клиента Paddle с соответствующим пользователем в базе данных вашего приложения. После завершения сеанса оформления заказа покупатель будет перенаправлен на специальную страницу успеха, где вы сможете отобразить покупателю информационное сообщение.

В представлении `buy` мы добавим кнопку для отображения оформления заказа. Компонент Blade `paddle-button` включен в состав Cashier Paddle; однако вы также можете [вручную отобразить оформления заказа](#manually-rendering-an-overlay-checkout):

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy Product
</x-paddle-button>
```

<a name="providing-meta-data-to-paddle-checkout"></a>
#### Предоставление метаданных для Paddle Checkout

При продаже товаров принято отслеживать выполненные заказы и приобретенные товары с помощью моделей `Cart` (корзина) и `Order` (заказ), определенных вашим собственным приложением. При перенаправлении клиентов на оформления заказа Paddle для совершения покупки вам может потребоваться предоставить существующий идентификатор заказа, чтобы вы могли связать завершенную покупку с соответствующим заказом, когда клиент будет перенаправлен обратно в ваше приложение.

Для этого вы можете предоставить массив пользовательских данных методу `checkout`. Давайте представим, что ожидающий `Order` создается в нашем приложении, когда пользователь начинает процесс оформления заказа. Помните, что модели `Cart` и `Order` в этом примере являются иллюстративными и не предоставлены Cashier. Вы можете свободно реализовать эти концепции в зависимости от потребностей вашего собственного приложения:

    use App\Models\Cart;
    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/cart/{cart}/checkout', function (Request $request, Cart $cart) {
        $order = Order::create([
            'cart_id' => $cart->id,
            'price_ids' => $cart->price_ids,
            'status' => 'incomplete',
        ]);

        $checkout = $request->user()->checkout($order->price_ids)
            ->customData(['order_id' => $order->id]);

        return view('billing', ['checkout' => $checkout]);
    })->name('checkout');

Как вы можете видеть в приведенном выше примере, когда пользователь начинает процесс оформления заказа, мы предоставим все идентификаторы цен Paddle, связанные с корзиной/заказом, методу `checkout`. Конечно, ваше приложение отвечает за связь этих товаров с «корзиной покупок» или заказом по мере их добавления покупателем. Мы также предоставляем идентификатор заказа Paddle Checkout Overlay с помощью метода `customData`.

Конечно, вы, скорее всего, захотите пометить заказ как «завершенный», как только покупатель завершит процесс оформления заказа. Для этого вы можете прослушивать веб-хуки, отправляемые Paddle и вызываемые через события Cashier, для хранения информации о заказе в вашей базе данных.

Для начала прослушайте событие `TransactionCompleted`, отправленное Cashier. Обычно вам следует зарегистрировать прослушиватель событий в методе `boot` `AppServiceProvider` вашего приложения:

    use App\Listeners\CompleteOrder;
    use Illuminate\Support\Facades\Event;
    use Laravel\Paddle\Events\TransactionCompleted;

    /**
     * Запуск любых служб приложений.
     */
    public function boot(): void
    {
        Event::listen(TransactionCompleted::class, CompleteOrder::class);
    }

В этом примере прослушиватель `CompleteOrder` может выглядеть следующим образом:

    namespace App\Listeners;

    use App\Models\Order;
    use Laravel\Paddle\Cashier;
    use Laravel\Paddle\Events\TransactionCompleted;

    class CompleteOrder
    {
        /**
         * Обработка входящего события веб-хука Cashier.
         */
        public function handle(TransactionCompleted $event): void
        {
            $orderId = $event->payload['data']['custom_data']['order_id'] ?? null;

            $order = Order::findOrFail($orderId);

            $order->update(['status' => 'completed']);
        }
    }

Пожалуйста, обратитесь к документации Paddle для получения дополнительной информации о [данных, содержащихся в событии `transaction.completed`](https://developer.paddle.com/webhooks/transactions/transaction-completed).

<a name="quickstart-selling-subscriptions"></a>
### Продажа подписок

> [!NOTE]
> Прежде чем использовать Paddle Checkout, вам следует определить продукты с фиксированными ценами на панели управления Paddle. Кроме того, вам следует [настроить обработку веб-хуков Paddle](#handling-paddle-webhooks).

Предложение выставления счетов за продукты и подписки через ваше приложение может напугать. Однако благодаря Cashier и [Paddle Checkout Overlay](https://www.paddle.com/billing/checkout) вы можете легко создать современную и надежную платежную интеграцию.

Чтобы узнать, как продавать подписки с помощью Cashier и Paddle Checkout Overlay, давайте рассмотрим простой сценарий службы подписки с базовым ежемесячным (`price_basic_monthly`) и годовым (`price_basic_yearly`) планом. Эти две цены можно сгруппировать под «Базовым» продуктом (`pro_basic`) на нашей панели управления Paddle. Кроме того, наша служба подписки может предлагать экспертный план под названием `pro_expert`.

Во-первых, давайте выясним, как клиент может подписаться на наши услуги. Конечно, вы можете себе представить, что клиент может нажать кнопку «подписаться» на базовый план на странице цен нашего приложения. Эта кнопка вызовет Paddle Checkout Overlay для выбранного плана. Для начала давайте инициируем сеанс оформления заказа с помощью метода `checkout`:

    use Illuminate\Http\Request;

    Route::get('/subscribe', function (Request $request) {
        $checkout = $request->user()->checkout('price_basic_monthly')
            ->returnTo(route('dashboard'));

        return view('subscribe', ['checkout' => $checkout]);
    })->name('subscribe');

В представлении `subscribe` мы добавим кнопку для оформления заказа. Компонент Blade `paddle-button` включен в состав Cashier Paddle; однако вы также можете [вручную отобразить оформление заказа](#manually-rendering-an-overlay-checkout):

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

Теперь, когда нажата кнопка «Подписаться», клиент сможет ввести свои платежные данные и инициировать подписку. Чтобы узнать, когда на самом деле началась подписка (поскольку для обработки некоторых способов оплаты требуется несколько секунд), вам также следует [настроить обработку веб-хука Cashier](#handling-paddle-webhooks).

Теперь, когда клиенты могут оформлять подписки, нам необходимо ограничить определенные части нашего приложения, чтобы доступ к ним могли получить только подписанные пользователи. Конечно, мы всегда можем определить текущий статус подписки пользователя с помощью метода `subscribed`, предоставляемого трейтом Cashier `Billable`:

```blade
@if ($user->subscribed())
    <p>You are subscribed.</p>
@endif
```

Мы даже можем легко определить, подписан ли пользователь на конкретный продукт или цену:

```blade
@if ($user->subscribedToProduct('pro_basic'))
    <p>You are subscribed to our Basic product.</p>
@endif

@if ($user->subscribedToPrice('price_basic_monthly'))
    <p>You are subscribed to our monthly Basic plan.</p>
@endif
```

<a name="quickstart-building-a-subscribed-middleware"></a>
#### Создание посредника по подписке

Для удобства вы можете создать [посредника](/docs/{{version}}/middleware), которое определяет, исходит ли входящий запрос от подписанного пользователя. После определения этого посредника вы можете легко назначить его маршруту, чтобы запретить пользователям, не подписанным на него, доступ к маршруту:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class Subscribed
    {
        /**
         * Обработка входящего запроса.
         */
        public function handle(Request $request, Closure $next): Response
        {
            if (! $request->user()?->subscribed()) {
                // Redirect user to billing page and ask them to subscribe...
                return redirect('/subscribe');
            }

            return $next($request);
        }
    }

После определения посредника вы можете назначить его маршруту:

    use App\Http\Middleware\Subscribed;

    Route::get('/dashboard', function () {
        // ...
    })->middleware([Subscribed::class]);

<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### Разрешение клиентам управлять своим тарифным планом

Конечно, клиенты могут захотеть изменить свой план подписки на другой продукт или «уровень». В нашем примере выше мы хотели бы позволить клиенту изменить свой план с ежемесячной подписки на годовую подписку. Для этого вам нужно реализовать что-то вроде кнопки, которая приведет к следующему маршруту:

    use Illuminate\Http\Request;

    Route::put('/subscription/{price}/swap', function (Request $request, $price) {
        $user->subscription()->swap($price); // В этом примере «$price» — «price_basic_yearly».

        return redirect()->route('dashboard');
    })->name('subscription.swap');

Помимо замены планов вам также необходимо разрешить своим клиентам отменить подписку. Как и при смене планов, предоставьте кнопку, которая ведет к следующему маршруту:

    use Illuminate\Http\Request;

    Route::put('/subscription/cancel', function (Request $request, $price) {
        $user->subscription()->cancel();

        return redirect()->route('dashboard');
    })->name('subscription.cancel');

И теперь ваша подписка будет отменена по окончании расчетного периода.

> [!NOTE]
> Пока вы настроили обработку веб-хука Cashier, Cashier будет автоматически синхронизировать таблицы базы данных, связанные с Cashier, путем проверки входящих веб-хуков из Paddle. Так, например, когда вы отменяете подписку клиента через панель управления Paddle, Cashier получит соответствующий вебхук и отметит подписку как «отмененную» в базе данных вашего приложения.

<a name="checkout-sessions"></a>
## Сессии оформления заказа

Большинство операций по выставлению счетов клиентам выполняются с использованием «оформление заказа» через [виджет Paddle Checkout Overlay](https://developer.paddle.com/build/checkout/build-overlay-checkout) или с помощью [встроенного оформления заказа](https://developer.paddle.com/build/checkout/build-branded-inline-checkout).

Прежде чем обрабатывать платежи при оформлении заказа с помощью Paddle, вам следует определить [ссылку для оплаты по умолчанию](https://developer.paddle.com/build/transactions/default-paying-link#set-default-link) вашего приложения на панели настроек оформления заказа Paddle.

<a name="overlay-checkout"></a>
### Всплывающий виджет оформления заказа

Прежде чем отображать всплывающий виджет оформления заказа, необходимо создать сеанс оформления заказа с помощью Cashier. Сеанс оформления заказа проинформирует виджет оформления заказа об операции выставления счета, которую необходимо выполнить:

    use Illuminate\Http\Request;

    Route::get('/buy', function (Request $request) {
        $checkout = $user->checkout('pri_34567')
            ->returnTo(route('dashboard'));

        return view('billing', ['checkout' => $checkout]);
    });

Кассир включает в себя [компонент Blade](/docs/{{version}}/blade#comComponents) `paddle-button`. Вы можете передать сеанс оформления заказа этому компоненту в качестве «реквизита». Затем, когда эта кнопка будет нажата, отобразится виджет оформления заказа Paddle:

```html
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

По умолчанию виджет будет отображаться с использованием стиля Paddle. Вы можете настроить виджет, добавив к компоненту [поддерживаемые атрибуты Paddle](https://developer.paddle.com/paddlejs/html-data-attributes), например атрибут `data-theme='light'`:

```html
<x-paddle-button :url="$payLink" class="px-8 py-4" data-theme="light">
    Subscribe
</x-paddle-button>
```

Виджет оформления заказа Paddle является асинхронным. Как только пользователь создаст подписку в виджете, Paddle отправит вашему приложению вебхук, чтобы вы могли правильно обновить состояние подписки в базе данных вашего приложения. Поэтому важно правильно [настроить веб-хуки](#handling-paddle-webhooks), чтобы учитывать изменения состояния из Paddle.

> [!WARNING]
> После изменения состояния подписки задержка получения соответствующего веб-хука обычно минимальна, но вы должны учитывать это в своем приложении, учитывая, что подписка вашего пользователя может быть недоступна сразу после завершения оформления заказа.

<a name="manually-rendering-an-overlay-checkout"></a>
#### Ручная визуализация всплывающего виджета оформления заказа

Вы также можете вручную визуализировать всплывающий виджет оформления заказа без использования встроенных компонентов Blade Laravel. Для начала создайте сеанс оформления заказа [как показано в предыдущих примерах](#overlay-checkout):

    use Illuminate\Http\Request;

    Route::get('/buy', function (Request $request) {
        $checkout = $user->checkout('pri_34567')
            ->returnTo(route('dashboard'));

        return view('billing', ['checkout' => $checkout]);
    });

Далее вы можете использовать Paddle.js для инициализации оформления заказа. В этом примере мы создадим ссылку, которой будет присвоен класс `paddle_button`. Paddle.js обнаружит этот класс и отобразит оформление заказа при нажатии ссылки:

```blade
<?php
$items = $checkout->getItems();
$customer = $checkout->getCustomer();
$custom = $checkout->getCustomData();
?>

<a
    href='#!'
    class='paddle_button'
    data-items='{!! json_encode($items) !!}'
    @if ($customer) data-customer-id='{{ $customer->paddle_id }}' @endif
    @if ($custom) data-custom-data='{{ json_encode($custom) }}' @endif
    @if ($returnUrl = $checkout->getReturnUrl()) data-success-url='{{ $returnUrl }}' @endif
>
    Buy Product
</a>
```

<a name="inline-checkout"></a>
### Встроенный виджет овормления заказа

Если вы не хотите использовать виджет Paddle оформления заказа в стиле «наложение», Paddle также предоставляет возможность отображать виджет встроенным. Хотя этот подход не позволяет вам настраивать какие-либо HTML-поля оформления заказа, он позволяет встроить виджет в ваше приложение.

Чтобы вам было проще начать работу со встроенной оплатой, Cashier включает в себя компонент Blade `paddle-checkout`. Чтобы начать, вам следует [создать сеанс оформления заказа](#overlay-checkout):

    use Illuminate\Http\Request;

    Route::get('/buy', function (Request $request) {
        $checkout = $user->checkout('pri_34567')
            ->returnTo(route('dashboard'));

        return view('billing', ['checkout' => $checkout]);
    });

Затем вы можете передать сеанс оформления заказа в атрибут `checkout` компонента:

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" />
```

Чтобы настроить высоту встроенного компонента оформления заказа, вы можете передать атрибут `height` компоненту Blade:

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" height="500" />
```

Пожалуйста, обратитесь к [руководству по Paddle Inline Checkout](https://developer.paddle.com/build/checkout/build-branded-inline-checkout) и [доступным настройкам оформления заказа](https://developer.paddle.com/build/checkout/set-up-checkout-default-settings) для получения дополнительной информации о параметрах настройки встроенной проверки.

<a name="manually-rendering-an-inline-checkout"></a>
#### Ручная визуализация встроенного оформления заказа

Вы также можете вручную визуализировать встроенное оформление заказа без использования встроенных компонентов Blade Laravel. Для начала создайте сеанс оформления заказа [как показано в предыдущих примерах](#inline-checkout):

    use Illuminate\Http\Request;

    Route::get('/buy', function (Request $request) {
        $checkout = $user->checkout('pri_34567')
            ->returnTo(route('dashboard'));

        return view('billing', ['checkout' => $checkout]);
    });

Далее вы можете использовать Paddle.js для инициализации оформления заказа. В этом примере мы продемонстрируем это, используя [Alpine.js](https://github.com/alpinejs/alpine); однако вы можете изменить этот пример для своего собственного фронтенда:

```blade
<?php
$options = $checkout->options();

$options['settings']['frameTarget'] = 'paddle-checkout';
$options['settings']['frameInitialHeight'] = 366;
?>

<div class="paddle-checkout" x-data="{}" x-init="
    Paddle.Checkout.open(@json($options));
">
</div>
```

<a name="guest-checkouts"></a>
### Гостевые оформления заказов

Иногда вам может потребоваться создать сеанс оформления заказа для пользователей, которым не нужна учетная запись в вашем приложении. Для этого вы можете использовать метод `guest`:

    use Illuminate\Http\Request;
    use Laravel\Paddle\Checkout;

    Route::get('/buy', function (Request $request) {
        $checkout = Checkout::guest('pri_34567')
            ->returnTo(route('home'));

        return view('billing', ['checkout' => $checkout]);
    });

Then, you may provide the checkout session to the [Paddle button](#overlay-checkout) or [inline checkout](#inline-checkout) Blade components.
Затем вы можете предоставить сеанс оформления заказа компонентам Blade [кнопка Paddle](#overlay-checkout) или [встроенное оформление заказа](#inline-checkout).

<a name="price-previews"></a>
## Предварительный просмотр цен

Paddle позволяет вам настраивать цены для каждой валюты, что, по сути, позволяет вам настраивать разные цены для разных стран. Cashier Paddle позволяет вам получить все эти цены, используя метод `previewPrices`. Этот метод принимает идентификаторы цен, для которых вы хотите получить цены:

    use Laravel\Paddle\Cashier;

    $prices = Cashier::previewPrices(['pri_123', 'pri_456']);

Валюта будет определена на основе IP-адреса запроса; однако вы можете дополнительно указать конкретную страну для получения цен:

    use Laravel\Paddle\Cashier;

    $prices = Cashier::previewPrices(['pri_123', 'pri_456'], ['address' => [
        'country_code' => 'BE',
        'postal_code' => '1234',
    ]]);

После получения цен вы можете отображать их по своему усмотрению:

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->total() }}</li>
    @endforeach
</ul>
```

Вы также можете отобразить промежуточную цену и сумму налога отдельно:

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->subtotal() }} (+ {{ $price->tax() }} tax)</li>
    @endforeach
</ul>
```

Для получения дополнительной информации [ознакомьтесь с документацией Paddle по API, касающейся предварительного просмотра цен](https://developer.paddle.com/api-reference/pricing-preview/preview-prices).

<a name="customer-price-previews"></a>
### Предварительный просмотр цен клиентов

Если пользователь уже является клиентом и вы хотите отобразить цены, применимые к этому клиенту, вы можете сделать это, получив цены непосредственно из экземпляра клиента:

    use App\Models\User;

    $prices = User::find(1)->previewPrices(['pri_123', 'pri_456']);

Внутри Cashier будет использовать идентификатор клиента пользователя для получения цен в его валюте. Так, например, пользователь, проживающий в США, увидит цены в долларах США, а пользователь из Бельгии увидит цены в евро. Если соответствующая валюта не найдена, будет использоваться валюта продукта по умолчанию. Вы можете настроить все цены на продукт или план подписки на панели управления Paddle.

<a name="price-discounts"></a>
### Скидки

Вы также можете выбрать отображение цен после скидки. При вызове метода `previewPrices` вы указываете идентификатор скидки с помощью опции `discount_id`:

    use Laravel\Paddle\Cashier;

    $prices = Cashier::previewPrices(['pri_123', 'pri_456'], [
        'discount_id' => 'dsc_123'
    ]);

Затем отобразите рассчитанные цены:

```blade
<ul>
    @foreach ($prices as $price)
        <li>{{ $price->product['name'] }} - {{ $price->total() }}</li>
    @endforeach
</ul>
```

<a name="customers"></a>
## Клиенты

<a name="customer-defaults"></a>
### Настройки клиента по умолчанию

Cashier позволяет вам определить некоторые полезные настройки по умолчанию для ваших клиентов при создании сеансов оформления заказа. Установка этих значений по умолчанию позволяет вам предварительно заполнить адрес электронной почты и имя клиента, чтобы он мог немедленно перейти к платежной части виджета оформления заказа. Вы можете установить эти значения по умолчанию, переопределив следующие методы в вашей оплатной модели:

    /**
     * Получение имя клиента, которое будет ассоциироваться с Paddle.
     */
    public function paddleName(): string|null
    {
        return $this->name;
    }

    /**
     * Получение адреса электронной почты клиента, чтобы связаться с Paddle.
     */
    public function paddleEmail(): string|null
    {
        return $this->email;
    }

Эти значения по умолчанию будут использоваться для каждого действия в Cashier, которое генерирует [сессию оформления заказа](#checkout-sessions).

<a name="retrieving-customers"></a>
### Привлечение клиентов

Вы можете получить клиента по его идентификатору клиента Paddle, используя метод `Cashier::findBillable`. Этот метод вернет экземпляр оплачиваемой модели:

    use Laravel\Paddle\Cashier;

    $user = Cashier::findBillable($customerId);

<a name="creating-customers"></a>
### Создание клиентов

Иногда вам может потребоваться создать клиента Paddle, не оформляя подписку. Вы можете сделать это, используя метод `createAsCustomer`:

    $customer = $user->createAsCustomer();

Возвращается экземпляр `Laravel\Paddle\Customer`. После того как клиент будет создан в Paddle, вы сможете начать подписку позже. Вы можете предоставить дополнительный массив `$options` для передачи любых дополнительных [параметров создания клиента, которые поддерживаются Paddle API](https://developer.paddle.com/api-reference/customers/create-customer):

    $customer = $user->createAsCustomer($options);

<a name="subscriptions"></a>
## Подписки

<a name="creating-subscriptions"></a>
### Создание подписок

Чтобы создать подписку, сначала извлеките экземпляр вашей оплачиваемой модели из базы данных, которая обычно представляет собой экземпляр `App\Models\User`. После того, как вы получили экземпляр модели, вы можете использовать метод `subscribe` для создания сеанса оформления заказа модели:

    use Illuminate\Http\Request;

    Route::get('/user/subscribe', function (Request $request) {
        $checkout = $request->user()->subscribe($premium = 12345, 'default')
            ->returnTo(route('home'));

        return view('billing', ['checkout' => $checkout]);
    });

Первый аргумент, передаваемый методу `subscribe`, — это конкретная цена, на которую подписывается пользователь. Это значение должно соответствовать идентификатору цены в Paddle. Метод `returnTo` принимает URL-адрес, на который будет перенаправлен ваш пользователь после успешного завершения оформления заказа. Второй аргумент, передаваемый методу `subscribe`, должен быть внутренним «типом» подписки. Если ваше приложение предлагает только одну подписку, вы можете назвать ее `default` или `primary`. Этот тип подписки предназначен только для внутреннего использования приложения и не предназначен для отображения пользователям. Кроме того, он не должен содержать пробелов и никогда не должен меняться после создания подписки.

Вы также можете предоставить массив пользовательских метаданных, касающихся подписки, используя метод `customData`:

    $checkout = $request->user()->subscribe($premium = 12345, 'default')
        ->customData(['key' => 'value'])
        ->returnTo(route('home'));

После создания сеанса оформления подписки он может быть передан в `paddle-button` [компонент Blade](#overlay-checkout), который включен в Cashier Paddle:

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

After the user has finished their checkout, a `subscription_created` webhook will be dispatched from Paddle. Cashier will receive this webhook and setup the subscription for your customer. In order to make sure all webhooks are properly received and handled by your application, ensure you have properly [setup webhook handling](#handling-paddle-webhooks).
После того, как пользователь завершит оформление заказа, из Paddle будет отправлен вебхук `subscription_created`. Cashier получит этот вебхук и настроит подписку для вашего клиента. Чтобы убедиться, что все веб-хуки правильно принимаются и обрабатываются вашим приложением, убедитесь, что у вас правильно настроена [настройка обработки веб-хуков](#handling-paddle-webhooks).

<a name="checking-subscription-status"></a>
### Проверка статуса подписки

Как только пользователь подпишется на ваше приложение, вы сможете проверить статус его подписки различными удобными способами. Во-первых, метод `subscribed` возвращает `true`, если у пользователя есть действующая подписка, даже если подписка в настоящее время находится в пробном периоде:

    if ($user->subscribed()) {
        // ...
    }

Если ваше приложение предлагает несколько подписок, вы можете указать подписку при вызове метода `subscribed`:

    if ($user->subscribed('default')) {
        // ...
    }

Метод `subscribed` также является отличным кандидатом на роль [псредника маршрута](/docs/{{version}}/middleware), позволяющего фильтровать доступ к маршрутам и контроллерам на основе статуса подписки пользователя:

    <?php

    namespace App\Http\Middleware;

    use Closure;
    use Illuminate\Http\Request;
    use Symfony\Component\HttpFoundation\Response;

    class EnsureUserIsSubscribed
    {
        /**
         * Обработка входящего запроса.
         *
         * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
         */
        public function handle(Request $request, Closure $next): Response
        {
            if ($request->user() && ! $request->user()->subscribed()) {
                // This user is not a paying customer...
                return redirect('/billing');
            }

            return $next($request);
        }
    }

Если вы хотите определить, находится ли у пользователя пробный период, вы можете использовать метод `onTrial`. Этот метод может быть полезен для определения того, следует ли отображать пользователю предупреждение о том, что у него все еще находится пробный период:

    if ($user->subscription()->onTrial()) {
        // ...
    }

Метод `subscribedToPrice` может использоваться для определения того, подписан ли пользователь на данный план, на основе данного идентификатора цены Paddle. В этом примере мы определим, активно ли оформлена подписка пользователя `default` по ежемесячной цене:

    if ($user->subscribedToPrice($monthly = 'pri_123', 'default')) {
        // ...
    }

Метод `recurring` можно использовать, чтобы определить, находится ли пользователь в настоящее время на активной подписке и больше не находится в пробном или льготном периоде:

    if ($user->subscription()->recurring()) {
        // ...
    }

<a name="canceled-subscription-status"></a>
#### Статус отмены подписки

Чтобы определить, был ли пользователь когда-то активным подписчиком, но отменил подписку, вы можете использовать метод `canceled`:

    if ($user->subscription()->canceled()) {
        // ...
    }

Вы также можете определить, отменил ли пользователь свою подписку, но у него все еще действует «льготный период» до полного истечения срока действия подписки. Например, если пользователь отменяет подписку 5 марта, срок действия которой первоначально должен был истечь 10 марта, для пользователя действует «льготный период» до 10 марта. Кроме того, метод `subscribed` все равно будет возвращать `true` в течение этого времени:

    if ($user->subscription()->onGracePeriod()) {
        // ...
    }

<a name="past-due-status"></a>
#### Статус просрочки платежа

Если платеж по подписке не выполнен, он будет помечен как `past_due`. Когда ваша подписка находится в этом состоянии, она не будет активна, пока клиент не обновит свою платежную информацию. Вы можете определить, просрочена ли подписка, используя метод `pastDue` в экземпляре подписки:

    if ($user->subscription()->pastDue()) {
        // ...
    }

Если срок подписки просрочен, вы должны поручить пользователю [обновить платежную информацию](#updating-pay-information).

Если вы хотите, чтобы подписки по-прежнему считались действительными, когда они истекли (`past_due`), вы можете использовать метод `keepPastDueSubscriptionsActive`, предоставляемый Cashier. Обычно этот метод следует вызывать в методе `register` вашего `AppServiceProvider`:

    use Laravel\Paddle\Cashier;

    /**
     * Зарегистрируйте любые службы приложений.
     */
    public function register(): void
    {
        Cashier::keepPastDueSubscriptionsActive();
    }

> [!WARNING]
> Когда подписка находится в состоянии `past_due`, ее нельзя изменить до тех пор, пока информация об оплате не будет обновлена. Таким образом, методы `swap` и `updateQuantity` выдадут исключение, когда подписка находится в состоянии `past_due`.

<a name="subscription-scopes"></a>
#### Области подписки

Большинство состояний подписки также доступны в виде областей запроса, поэтому вы можете легко запросить в базе данных подписки, находящиеся в заданном состоянии:

    // Получить все действительные подписки...
    $subscriptions = Subscription::query()->valid()->get();

    // Получить все отмененные подписки пользователя...
    $subscriptions = $user->subscriptions()->canceled()->get();

Полный список доступных областей доступен ниже:

    Subscription::query()->valid();
    Subscription::query()->onTrial();
    Subscription::query()->expiredTrial();
    Subscription::query()->notOnTrial();
    Subscription::query()->active();
    Subscription::query()->recurring();
    Subscription::query()->pastDue();
    Subscription::query()->paused();
    Subscription::query()->notPaused();
    Subscription::query()->onPausedGracePeriod();
    Subscription::query()->notOnPausedGracePeriod();
    Subscription::query()->canceled();
    Subscription::query()->notCanceled();
    Subscription::query()->onGracePeriod();
    Subscription::query()->notOnGracePeriod();

<a name="subscription-single-charges"></a>
### Разовые платежи за подписку

Единая плата за подписку позволяет взимать с подписчиков единовременную плату в дополнение к их подписке. При вызове метода `charge` вы должны указать один или несколько идентификаторов цены:

    // Установить одну цену...
    $response = $user->subscription()->charge('pri_123');

    // Назначить несколько цен одновременно...
    $response = $user->subscription()->charge(['pri_123', 'pri_456']);

Метод `charge` фактически не взимает с клиента плату до следующего периода выставления счетов за его подписку. Если вы хотите немедленно выставить счет клиенту, вместо этого вы можете использовать метод `chargeAndInvoice`:

    $response = $user->subscription()->chargeAndInvoice('pri_123');

<a name="updating-payment-information"></a>
### Обновление платежной информации

Paddle всегда сохраняет способ оплаты для каждой подписки. Если вы хотите обновить метод оплаты по умолчанию для подписки, вам следует перенаправить своего клиента на страницу обновления размещенного метода оплаты Paddle, используя метод `redirectToUpdatePaymentMethod` в модели подписки:

    use Illuminate\Http\Request;

    Route::get('/update-payment-method', function (Request $request) {
        $user = $request->user();

        return $user->subscription()->redirectToUpdatePaymentMethod();
    });

Когда пользователь завершит обновление своей информации, Paddle отправит веб-хук `subscription_updated`, и сведения о подписке будут обновлены в базе данных вашего приложения.

<a name="changing-plans"></a>
### Изменение планов

После того как пользователь подписался на ваше приложение, он может иногда захотеть перейти на новый план подписки. Чтобы обновить план подписки для пользователя, вам необходимо передать идентификатор цены Paddle в метод `swap`:

    use App\Models\User;

    $user = User::find(1);

    $user->subscription()->swap($premium = 'pri_456');

Если вы хотите поменять планы и немедленно выставить пользователю счет вместо того, чтобы ждать следующего платежного цикла, вы можете использовать метод `swapAndInvoice`:

    $user = User::find(1);

    $user->subscription()->swapAndInvoice($premium = 'pri_456');

<a name="prorations"></a>
#### Пропорции

По умолчанию Paddle пропорционально распределяет расходы при переключении между планами. Метод `noProrate` можно использовать для обновления подписок без пропорционального распределения расходов:

    $user->subscription('default')->noProrate()->swap($premium = 'pri_456');

Если вы хотите немедленно отключить пропорциональное распределение и выставлять счета клиентам, вы можете использовать метод `swapAndInvoice` в сочетании с `noProrate`:

    $user->subscription('default')->noProrate()->swapAndInvoice($premium = 'pri_456');

Или, чтобы не выставлять счет клиенту за изменение подписки, вы можете использовать метод `doNotBill`:

    $user->subscription('default')->doNotBill()->swap($premium = 'pri_456');

Для получения дополнительной информации о политике пропорционального распределения Paddle обратитесь к [документации по пропорциональному распределению](https://developer.paddle.com/concepts/subscriptions/proration).

<a name="subscription-quantity"></a>
### Количество подписки

Иногда на подписки влияет «количество». Например, приложение для управления проектами может взимать 10 долларов США в месяц за проект. Чтобы легко увеличить или уменьшить количество вашей подписки, используйте методы `incrementQuantity` и `decrementQuantity`:

    $user = User::find(1);

    $user->subscription()->incrementQuantity();

    // Добавить пять к текущему количеству подписки...
    $user->subscription()->incrementQuantity(5);

    $user->subscription()->decrementQuantity();

    // Вычесть пять из текущего количества подписки...
    $user->subscription()->decrementQuantity(5);

Альтернативно вы можете установить определенное количество, используя метод `updateQuantity`:

    $user->subscription()->updateQuantity(10);

Метод `noProrate` можно использовать для обновления количества подписки без пропорционального распределения расходов:

    $user->subscription()->noProrate()->updateQuantity(10);

<a name="quantities-for-subscription-with-multiple-products"></a>
#### Количество для подписок с несколькими продуктами

Если ваша подписка представляет собой [подписку с несколькими продуктами](#subscriptions-with-multiple-products), вам следует передать идентификатор цены, количество которой вы хотите увеличить или уменьшить, в качестве второго аргумента методов увеличения/уменьшения:

    $user->subscription()->incrementQuantity(1, 'price_chat');

<a name="subscriptions-with-multiple-products"></a>
### Подписки с несколькими продуктами

[Подписка на несколько продуктов](https://developer.paddle.com/build/subscriptions/add-remove-products-prices-addons) позволяет назначать несколько продуктов для выставления счетов одной подписке. Например, представьте, что вы создаете приложение «службы поддержки» для обслуживания клиентов, базовая цена подписки которого составляет 10 долларов в месяц, но предлагает дополнительный продукт для живого чата за дополнительные 15 долларов в месяц.

При создании сеансов оформления подписки вы можете указать несколько продуктов для данной подписки, передав массив цен в качестве первого аргумента методу `subscribe`:

    use Illuminate\Http\Request;

    Route::post('/user/subscribe', function (Request $request) {
        $checkout = $request->user()->subscribe([
            'price_monthly',
            'price_chat',
        ]);

        return view('billing', ['checkout' => $checkout]);
    });

В приведенном выше примере к подписке `default` для клиента будут прикреплены две цены. Обе цены будут взиматься в соответствующие расчетные периоды. При необходимости вы можете передать ассоциативный массив пар ключ/значение, чтобы указать конкретное количество для каждой цены:

    $user = User::find(1);

    $checkout = $user->subscribe('default', ['price_monthly', 'price_chat' => 5]);

Если вы хотите добавить еще одну цену к существующей подписке, вы должны использовать метод подписки `swap`. При вызове метода `swap` вы также должны указать текущие цены и количество подписки:

    $user = User::find(1);

    $user->subscription()->swap(['price_chat', 'price_original' => 2]);

В приведенном выше примере будет добавлена ​​новая цена, но клиенту не будет выставлен счет за нее до следующего платежного цикла. Если вы хотите немедленно выставить счет клиенту, вы можете использовать метод `swapAndInvoice`:

    $user->subscription()->swapAndInvoice(['price_chat', 'price_original' => 2]);

Вы можете удалить цены из подписок, используя метод `swap`, опустив цену, которую хотите удалить:

    $user->subscription()->swap(['price_original' => 2]);

> [!WARNING]
> Вы не можете удалить последнюю цену подписки. Вместо этого вам следует просто отменить подписку.

<a name="multiple-subscriptions"></a>
### Множественные подписки

Paddle позволяет вашим клиентам иметь несколько подписок одновременно. Например, вы можете управлять тренажерным залом, который предлагает подписку на плавание и подписку на тяжелую атлетику, и каждая подписка может иметь разные цены. Конечно, клиенты должны иметь возможность подписаться на один или оба плана.

Когда ваше приложение создает подписки, вы можете указать тип подписки методу `subscribe` в качестве второго аргумента. Типом может быть любая строка, представляющая тип подписки, которую инициирует пользователь:

    use Illuminate\Http\Request;

    Route::post('/swimming/subscribe', function (Request $request) {
        $checkout = $request->user()->subscribe($swimmingMonthly = 'pri_123', 'swimming');

        return view('billing', ['checkout' => $checkout]);
    });

В этом примере мы инициировали ежемесячную подписку на плавание для клиента. Однако позже они могут захотеть перейти на годовую подписку. При настройке подписки клиента мы можем просто поменять цену на подписку `swimming`:

    $user->subscription('swimming')->swap($swimmingYearly = 'pri_456');

Конечно, вы также можете полностью отменить подписку:

    $user->subscription('swimming')->cancel();

<a name="pausing-subscriptions"></a>
### Приостановка подписок

Чтобы приостановить подписку, вызовите метод `pause` в подписке пользователя:

    $user->subscription()->pause();

Когда подписка приостановлена, Cashier автоматически установит столбец `paused_at` в вашей базе данных. Этот столбец используется для определения того, когда метод `paused` должен начать возвращать `true`. Например, если клиент приостанавливает подписку 1 марта, но возобновление подписки не запланировано до 5 марта, метод `paused` будет продолжать возвращать `false` до 5 марта. Это связано с тем, что пользователю обычно разрешается продолжать использовать приложение до конца платежного цикла.

По умолчанию приостановка происходит в следующем интервале выставления счетов, чтобы клиент мог использовать оставшуюся часть оплаченного периода. Если вы хотите немедленно приостановить подписку, вы можете использовать метод `pauseNow`:

    $user->subscription()->pauseNow();

Используя метод `pauseUntil`, вы можете приостановить подписку до определенного момента времени:

    $user->subscription()->pauseUntil(now()->addMonth());

Или вы можете использовать метод `pauseNowUntil`, чтобы немедленно приостановить подписку до заданного момента времени:

    $user->subscription()->pauseNowUntil(now()->addMonth());

Вы можете определить, приостановил ли пользователь свою подписку, но все еще находится в «льготном периоде», используя метод `onPausedGracePeriod`:

    if ($user->subscription()->onPausedGracePeriod()) {
        // ...
    }

Чтобы возобновить приостановленную подписку, вы можете вызвать метод `resume` подписки:

    $user->subscription()->resume();

> [!WARNING]
> Подписку нельзя изменить, пока она приостановлена. Если вы хотите перейти на другой план или обновить количество, сначала необходимо возобновить подписку.

<a name="canceling-subscriptions"></a>
### Отмена подписок

Чтобы отменить подписку, вызовите метод `cancel` для подписки пользователя:

    $user->subscription()->cancel();

При отмене подписки Cashier автоматически установит столбец `ends_at` в вашей базе данных. Этот столбец используется для определения того, когда метод `subscribed` должен начать возвращать `false`. Например, если клиент отменяет подписку 1 марта, но ее завершение не планировалось до 5 марта, метод `subscribed` будет продолжать возвращать `true` до 5 марта. Это сделано потому, что пользователю обычно разрешается продолжать использовать приложение до конца платежного цикла.

Вы можете определить, отменил ли пользователь свою подписку, но все еще имеет «льготный период», используя метод `onGracePeriod`:

    if ($user->subscription()->onGracePeriod()) {
        // ...
    }

Если вы хотите немедленно отменить подписку, вы можете вызвать метод `cancelNow` для подписки:

    $user->subscription()->cancelNow();

Чтобы предотвратить отмену подписки в льготный период, вы можете вызвать метод `stopCancelation`:

    $user->subscription()->stopCancelation();

> [!WARNING]
> Подписки Paddle не могут быть возобновлены после отмены. Если ваш клиент желает возобновить подписку, ему придется создать новую подписку.

<a name="subscription-trials"></a>
## Пробная подписка

<a name="with-payment-method-up-front"></a>
### Со способом оплаты заранее

Если вы хотите предложить своим клиентам пробные периоды, одновременно собирая информацию о методах оплаты заранее, вам следует установить пробное время на панели управления Paddle по цене, на которую подписывается ваш клиент. Затем запустите сеанс оформления заказа как обычно:

    use Illuminate\Http\Request;

    Route::get('/user/subscribe', function (Request $request) {
        $checkout = $request->user()->subscribe('pri_monthly')
                    ->returnTo(route('home'));

        return view('billing', ['checkout' => $checkout]);
    });

Когда ваше приложение получит событие `subscription_created`, Cashier установит дату окончания пробного периода в записи подписки в базе данных вашего приложения, а также даст указание Paddle не начинать выставлять счета клиенту до истечения этой даты.

> [!WARNING]
> Если подписка клиента не отменена до даты окончания пробной версии, с нее будет снята плата, как только истечет пробная версия, поэтому вам следует обязательно уведомить своих пользователей о дате окончания пробной версии.

Вы можете определить, находится ли у пользователя пробный период, используя метод `onTrial` экземпляра пользователя или метод `onTrial` экземпляра подписки. Два приведенных ниже примера эквивалентны:

    if ($user->onTrial()) {
        // ...
    }

    if ($user->subscription()->onTrial()) {
        // ...
    }

Чтобы определить, истек ли срок действия существующей пробной версии, вы можете использовать методы `hasExpiredTrial`:

    if ($user->hasExpiredTrial()) {
        // ...
    }

    if ($user->subscription()->hasExpiredTrial()) {
        // ...
    }

Чтобы определить, находится ли пользователь на пробной версии для определенного типа подписки, вы можете указать тип методам `onTrial` или `hasExpiredTrial`:

    if ($user->onTrial('default')) {
        // ...
    }

    if ($user->hasExpiredTrial('default')) {
        // ...
    }

<a name="without-payment-method-up-front"></a>
### Без способа оплаты заранее

Если вы хотите предлагать пробные периоды без предварительного сбора информации о способе оплаты пользователя, вы можете установить в столбце `trial_ends_at` в записи клиента, прикрепленной к вашему пользователю, желаемую дату окончания пробного периода. Обычно это делается во время регистрации пользователя:

    use App\Models\User;

    $user = User::create([
        // ...
    ]);

    $user->createAsCustomer([
        'trial_ends_at' => now()->addDays(10)
    ]);

Cashier называет этот тип пробной версии «общей пробной версией», поскольку она не привязана ни к одной существующей подписке. Метод `onTrial` экземпляра `User` вернет `true`, если текущая дата не превышает значения `trial_ends_at`:

    if ($user->onTrial()) {
        // У пользователя действует пробный период...
    }

Когда вы будете готовы создать реальную подписку для пользователя, вы можете использовать метод `subscribe` как обычно:

    use Illuminate\Http\Request;

    Route::get('/user/subscribe', function (Request $request) {
        $checkout = $user->subscribe('pri_monthly')
            ->returnTo(route('home'));

        return view('billing', ['checkout' => $checkout]);
    });

Чтобы получить дату окончания пробной версии пользователя, вы можете использовать метод `trialEndsAt`. Этот метод вернет экземпляр даты Carbon, если пользователь находится на пробной версии, или `null`, если нет. Вы также можете передать необязательный параметр типа подписки, если хотите получить дату окончания пробной версии для конкретной подписки, отличной от стандартной:

    if ($user->onTrial('default')) {
        $trialEndsAt = $user->trialEndsAt();
    }

Вы можете использовать метод `onGenericTrial`, если хотите точно знать, что у пользователя находится «общий» пробный период и он еще не создал фактическую подписку:

    if ($user->onGenericTrial()) {
        // У пользователя действует «общий» пробный период...
    }

<a name="extend-or-activate-a-trial"></a>
### Продление или активировация пробной версии

Вы можете продлить существующий пробный период подписки, вызвав метод `extendTrial` и указав момент времени, когда пробная версия должна закончиться:

    $user->subscription()->extendTrial(now()->addDays(5));

Или вы можете немедленно активировать подписку, завершив ее пробную версию, вызвав метод `activate` подписки:

    $user->subscription()->activate();

<a name="handling-paddle-webhooks"></a>
## Обработка веб-хуков Paddle

Paddle может уведомлять ваше приложение о различных событиях через веб-хуки. По умолчанию маршрут, указывающий на контроллер веб-хука Cashier, регистрируется поставщиком услуг Cashier. Этот контроллер будет обрабатывать все входящие запросы веб-хука.

По умолчанию этот контроллер автоматически обрабатывает отмену подписок со слишком большим количеством неудачных платежей, обновлений подписок и изменений способов оплаты; однако, как мы скоро обнаружим, вы можете расширить этот контроллер для обработки любого события веб-хука Paddle, которое вам нравится.

To ensure your application can handle Paddle webhooks, be sure to [configure the webhook URL in the Paddle control panel](https://vendors.paddle.com/alerts-webhooks). By default, Cashier's webhook controller responds to the `/paddle/webhook` URL path. The full list of all webhooks you should enable in the Paddle control panel are:
Чтобы ваше приложение могло обрабатывать веб-хуки Paddle, обязательно [настройте URL-адрес веб-перехватчика на панели управления Paddle](https://vendors.paddle.com/alerts-webhooks). По умолчанию контроллер веб-хука Cashier отвечает на URL-путь `/paddle/webhook`. Полный список всех веб-хуков, которые следует включить в панели управления Paddle:

- Customer Updated
- Transaction Completed
- Transaction Updated
- Subscription Created
- Subscription Updated
- Subscription Paused
- Subscription Canceled

> [!WARNING]
> Make sure you protect incoming requests with Cashier's included [webhook signature verification](/docs/{{version}}/cashier-paddle#verifying-webhook-signatures) middleware.
> Убедитесь, что вы защищаете входящие запросы с помощью включенного в Cashier посредника [проверки подписи веб-хука](/docs/{{version}}/cashier-paddle#verifying-webhook-signatures).

<a name="webhooks-csrf-protection"></a>
#### Вебхуки и защита CSRF

Поскольку веб-хукам Paddle необходимо обходить [защиту CSRF](/docs/{{version}}/csrf) Laravel, вам следует убедиться, что Laravel не пытается проверить токен CSRF для входящих веб-хуков Paddle. Для этого вам следует исключить `paddle/*` из защиты CSRF в файле `bootstrap/app.php` вашего приложения:

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->validateCsrfTokens(except: [
            'paddle/*',
        ]);
    })

<a name="webhooks-local-development"></a>
#### Вебхуки и локальная разработка

Чтобы Paddle мог отправлять веб-хуки вашего приложения во время локальной разработки, вам необходимо предоставить доступ к вашему приложению через службу совместного использования сайтов, например [Ngrok](https://ngrok.com/) или [Expose](https://expose.dev/docs/introduction). Если вы разрабатываете свое приложение локально с помощью [Laravel Sail](/docs/{{version}}/sail), вы можете использовать команду Sail [команду общего доступа к сайту](/docs/{{version}}/sail#sharing-your-site).

<a name="defining-webhook-event-handlers"></a>
### Определение обработчиков событий веб-хука

Cashier автоматически обрабатывает отмену подписки при неудачных платежах и других распространенных веб-хуках Paddle. Однако если у вас есть дополнительные события веб-хука, которые вы хотели бы обработать, вы можете сделать это, прослушивая следующие события, отправляемые Cashier:

- `Laravel\Paddle\Events\WebhookReceived`
- `Laravel\Paddle\Events\WebhookHandled`

Оба события содержат полную полезную нагрузку веб-хука Paddle. Например, если вы хотите обрабатывать веб-хук `transaction.billed`, вы можете зарегистрировать [прослушиватель](/docs/{{version}}/events#defining-listeners), который будет обрабатывать событие:

    <?php

    namespace App\Listeners;

    use Laravel\Paddle\Events\WebhookReceived;

    class PaddleEventListener
    {
        /**
         * Handle получил веб-хуки Paddle.
         */
        public function handle(WebhookReceived $event): void
        {
            if ($event->payload['event_type'] === 'transaction.billed') {
                // Обработать входящее событие...
            }
        }
    }

Касса также генерирует события, посвященные типу полученного вебхука. В дополнение к полной полезной нагрузке Paddle они также содержат соответствующие модели, которые использовались для обработки веб-перехватчика, такие как оплачиваемая модель, подписка или квитанция:

- `Laravel\Paddle\Events\CustomerUpdated`
- `Laravel\Paddle\Events\TransactionCompleted`
- `Laravel\Paddle\Events\TransactionUpdated`
- `Laravel\Paddle\Events\SubscriptionCreated`
- `Laravel\Paddle\Events\SubscriptionUpdated`
- `Laravel\Paddle\Events\SubscriptionPaused`
- `Laravel\Paddle\Events\SubscriptionCanceled`

Вы также можете переопределить встроенный маршрут веб-хука по умолчанию, определив переменную среды `CASHIER_WEBHOOK` в файле `.env` вашего приложения. Это значение должно быть полным URL-адресом вашего маршрута веб-хука и должно совпадать с URL-адресом, установленным на панели управления Paddle:

```ini
CASHIER_WEBHOOK=https://example.com/my-paddle-webhook-url
```

<a name="verifying-webhook-signatures"></a>
### Проверка подписей вебхуков

Чтобы защитить свои веб-хуки, вы можете использовать [подписи веб-хуков Paddle](https://developer.paddle.com/webhook-reference/verifying-webhooks). Для удобства Cashier автоматически включает посредника, который проверяет правильность входящего запроса веб-хука Paddle.

Чтобы включить проверку веб-хука, убедитесь, что переменная среды `PADDLE_WEBHOOK_SECRET` определена в файле `.env` вашего приложения. Секрет вебхука можно получить на панели управления вашей учетной записи Paddle.

<a name="single-charges"></a>
## Разовые сборы

<a name="charging-for-products"></a>
### Плата за продукты

Если вы хотите инициировать покупку продукта для клиента, вы можете использовать метод `checkout` в экземпляре оплачиваемой модели, чтобы создать сеанс оформления заказа для покупки. Метод `checkout` принимает один или несколько идентификаторов цены. При необходимости можно использовать ассоциативный массив для указания количества приобретаемого продукта:

    use Illuminate\Http\Request;

    Route::get('/buy', function (Request $request) {
        $checkout = $request->user()->checkout(['pri_tshirt', 'pri_socks' => 5]);

        return view('buy', ['checkout' => $checkout]);
    });

После создания сеанса оформления заказа вы можете использовать предоставленную Cashier `paddle-button` [компонент Blade](#overlay-checkout), чтобы позволить пользователю просмотреть виджет оформления заказа Paddle и завершить покупку:

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Buy
</x-paddle-button>
```

Сеанс оформления заказа имеет метод `customData`, позволяющий передавать любые пользовательские данные, которые вы хотите, в базовую транзакцию. Пожалуйста, обратитесь к [документации Paddle](https://developer.paddle.com/build/transactions/custom-data), чтобы узнать больше о параметрах, доступных вам при передаче пользовательских данных:

    $checkout = $user->checkout('pri_tshirt')
        ->customData([
            'custom_option' => $value,
        ]);

<a name="refunding-transactions"></a>
### Возвратные транзакции

Транзакции возврата вернут сумму используя способ оплаты вашего клиента, который совершил покупку. Если вам необходимо вернуть деньги за покупку Paddle, вы можете использовать метод `refund` в модели `Cashier\Paddle\Transaction`. Этот метод принимает причину в качестве первого аргумента, один или несколько идентификаторов цен для возврата с необязательными суммами в виде ассоциативного массива. Вы можете получить транзакции для конкретной оплачиваемой модели, используя метод `transactions`.

Например, представьте, что мы хотим возместить определенную транзакцию по ценам `pri_123` и `pri_456`. Мы хотим полностью вернуть стоимость `pri_123`, но вернуть только два доллара за `pri_456`:

    use App\Models\User;

    $user = User::find(1);

    $transaction = $user->transactions()->first();

    $response = $transaction->refund('Accidental charge', [
        'pri_123', // Полностью верните эту цену...
        'pri_456' => 200, // Только частично возместите эту цену...
    ]);

В приведенном выше примере происходит возврат средств по определенным позициям в транзакции. Если вы хотите вернуть всю транзакцию, просто укажите причину:

    $response = $transaction->refund('Accidental charge');

Для получения дополнительной информации о возврате средств обратитесь к [документации по возврату средств Paddle](https://developer.paddle.com/build/transactions/create-transaction-adjustments).

> [!WARNING]
> Возврат всегда должен быть одобрен Paddle до полной обработки.

<a name="crediting-transactions"></a>
### Кредитные операции

Как и при возврате средств, вы также можете кредитовать транзакции. Зачисление транзакций добавит средства на баланс клиента, чтобы их можно было использовать для будущих покупок. Транзакции кредитования могут выполняться только для транзакций, собираемых вручную, но не для транзакций, собираемых автоматически (например, подписок), поскольку Paddle автоматически обрабатывает кредиты по подписке:

    $transaction = $user->transactions()->first();

    // Credit a specific line item fully...
    $response = $transaction->credit('Compensation', 'pri_123');

Для получения дополнительной информации [см. документацию Paddle по кредитованию](https://developer.paddle.com/build/transactions/create-transaction-adjustments).

> [!WARNING]
> Кредиты можно применять только для транзакций, собранных вручную. Автоматически собранные транзакции зачисляются самим Paddle.

<a name="transactions"></a>
## Транзакции

Вы можете легко получить массив транзакций оплачиваемой модели с помощью свойства `transactions`:

    use App\Models\User;

    $user = User::find(1);

    $transactions = $user->transactions;

Транзакции представляют собой оплату ваших продуктов и покупок и сопровождаются счетами. В базе данных вашего приложения сохраняются только завершенные транзакции.

При перечислении транзакций для клиента вы можете использовать методы экземпляра транзакции для отображения соответствующей платежной информации. Например, вы можете перечислить каждую транзакцию в таблице, чтобы пользователь мог легко загрузить любой из счетов:

```html
<table>
    @foreach ($transactions as $transaction)
        <tr>
            <td>{{ $transaction->billed_at->toFormattedDateString() }}</td>
            <td>{{ $transaction->total() }}</td>
            <td>{{ $transaction->tax() }}</td>
            <td><a href="{{ route('download-invoice', $transaction->id) }}" target="_blank">Download</a></td>
        </tr>
    @endforeach
</table>
```

Маршрут `download-invoice` может выглядеть следующим образом:

    use Illuminate\Http\Request;
    use Laravel\Paddle\Transaction;

    Route::get('/download-invoice/{transaction}', function (Request $request, Transaction $transaction) {
        return $transaction->redirectToInvoicePdf();
    })->name('download-invoice');

<a name="past-and-upcoming-payments"></a>
### Прошлые и предстоящие платежи

Вы можете использовать методы `lastPayment` и `nextPayment` для получения и отображения прошлых или предстоящих платежей клиента за повторяющиеся подписки:

    use App\Models\User;

    $user = User::find(1);

    $subscription = $user->subscription();

    $lastPayment = $subscription->lastPayment();
    $nextPayment = $subscription->nextPayment();

Оба эти метода вернут экземпляр `Laravel\Paddle\Payment`; однако `lastPayment` вернет `null`, если транзакции ещё не синхронизированы с помощью веб-хуков, а `nextPayment` вернет `null`, когда платежный цикл закончился (например, когда подписка была отменена):

```blade
Next payment: {{ $nextPayment->amount() }} due on {{ $nextPayment->date()->format('d/m/Y') }}
```

<a name="testing"></a>
## Тестирование

Во время тестирования вам следует вручную протестировать процесс выставления счетов, чтобы убедиться, что интеграция работает должным образом.

Для автоматических тестов, в том числе выполняемых в среде CI, вы можете использовать [HTTP-клиент Laravel](/docs/{{version}}/http-client#testing) для имитации HTTP-вызовов, сделанных в Paddle. Хотя при этом не проверяются фактические ответы Paddle, он дает возможность протестировать ваше приложение без фактического вызова API Paddle.
