---
git: 667396a7971720b717079436ff1fe8682168d72e
---


# Laravel Cashier (Stripe)

<a name="introduction"></a>
## Введение

[Laravel Cashier Stripe](https://github.com/laravel/cashier-stripe) предоставляет выразительный, понятный интерфейс для работы с услугами подписочного выставления счетов [Stripe](https://stripe.com). Он обрабатывает практически всю рутиноную часть кода для выставления счетов за подписку, которую вы не хотите писать. Кроме базового управления подпиской, Cashier может работать с купонами, изменять подписки, "количества" подписок, периоды благоприятного завершения подписки, а также генерировать счета в формате PDF.

<a name="upgrading-cashier"></a>
## Обновление Cashier

При обновлении до новой версии Cashier важно внимательно ознакомиться с [руководством по обновлению](https://github.com/laravel/cashier-stripe/blob/master/UPGRADE.md).

> [!WARNING]
> Чтобы избежать нарушений, Cashier использует фиксированную версию API Stripe. Cashier 15 использует версию API Stripe `2023-10-16`. Версия API Stripe будет обновляться в минорных релизах для использования новых функций и улучшений Stripe.

<a name="installation"></a>
## Установка

Сначала установите пакет Cashier для Stripe с помощью менеджера пакетов Composer:

```shell
composer require laravel/cashier
```

После установки пакета опубликуйте миграции Cashier с помощью команды Artisan `vendor:publish`:

```shell
php artisan vendor:publish --tag="cashier-migrations"
```

Затем перенесите базу данных:

```shell
php artisan migrate
```

Миграции Cashier добавят несколько столбцов в вашу таблицу `users`. Они также создадут новую таблицу `subscriptions`, чтобы хранить все подписки ваших клиентов, а также таблицу `subscription_items` для подписок с несколькими ценами.

Если вы хотите, вы также можете опубликовать файл конфигурации Cashier с помощью команды Artisan `vendor:publish`:

```shell
php artisan vendor:publish --tag="cashier-config"
```

Наконец, чтобы гарантировать правильную обработку всех событий Stripe Cashier, не забудьте [настроить обработку веб-хуков Cashier](#handling-stripe-webhooks).

> [!WARNING]
> Stripe рекомендует, чтобы любой столбец, используемый для хранения идентификаторов Stripe, был чувствителен к регистру. Поэтому вы должны убедиться, что сопоставление(collation) столбца `stripe_id` установлена в `utf8_bin` при использовании MySQL. Более подробную информацию об этом можно найти в [документации Stripe](https://stripe.com/docs/upgrades#what-changes-does-stripe-consider-to-be-backwards-compatible).

<a name="configuration"></a>
## Настройка

<a name="billable-model"></a>
### Модель с возможностью выставления счетов

Перед использованием Cashier добавьте трейт `Billable` к определению вашей модели с возможностью выставления счетов. Обычно это модель `App\Models\User`. Этот трейт предоставляет различные методы, позволяющие выполнять обычные задачи по выставлению счетов, такие как создание подписок, применение купонов и обновление информации о способе оплаты:

    use Laravel\Cashier\Billable;

    class User extends Authenticatable
    {
        use Billable;
    }

Cashier предполагает, что ваша модель с возможностью выставления счетов будет классом `App\Models\User`, который поставляется с Laravel. Если вы хотите изменить это, вы можете указать другую модель с помощью метода `useCustomerModel`. Его обычно следует вызывать в методе `boot` вашего класса `AppServiceProvider`:

    use App\Models\Cashier\User;
    use Laravel\Cashier\Cashier;

    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Cashier::useCustomerModel(User::class);
    }

> [!WARNING]
> Если вы используете модель, отличную от предоставляемой Laravel модели `App\Models\User`, вам потребуется опубликовать и изменить [миграции Cashier](#installation), чтобы они соответствовали названию таблицы вашей альтернативной модели.

<a name="api-keys"></a>
### Ключи API

Затем вам следует сконфигурировать ваши ключи API Stripe в файле `.env` вашего приложения. Вы можете получить ваши ключи API Stripe из панели управления Stripe:

```ini
STRIPE_KEY=your-stripe-key
STRIPE_SECRET=your-stripe-secret
STRIPE_WEBHOOK_SECRET=your-stripe-webhook-secret
```

> [!WARNING]
> Убедитесь, что переменная окружения `STRIPE_WEBHOOK_SECRET` определена в файле `.env` вашего приложения, поскольку эта переменная используется для обеспечения того, что входящие веб-хуки действительно от Stripe.

<a name="currency-configuration"></a>
### Настройка валюты

Cashier использует доллары США (USD) в качестве валюты по умолчанию. Вы можете изменить валюту по умолчанию, установив переменную окружения `CASHIER_CURRENCY` в файле `.env` вашего приложения:

```ini
CASHIER_CURRENCY=eur
```

В дополнение к настройке валюты Cashier, вы также можете указать язык, который будет использоваться при форматировании денежных значений для отображения в счетах. Внутренне Cashier использует [класс PHP `NumberFormatter`](https://www.php.net/manual/en/class.numberformatter.php) для установки языкового стандарта валюты:

```ini
CASHIER_CURRENCY_LOCALE=nl_BE
```

> [!WARNING]
> Чтобы использовать локали, отличные от `en`, убедитесь, что на вашем сервере установлено и настроено расширение PHP `ext-intl`.

<a name="tax-configuration"></a>
### Настройка налогов

Благодаря [Stripe Tax](https://stripe.com/tax), можно автоматически рассчитать налоги для всех счетов, сгенерированных Stripe. Вы можете включить автоматический расчет налогов, вызвав метод `calculateTaxes` в методе `boot` класса `App\Providers\AppServiceProvider` вашего приложения:

    use Laravel\Cashier\Cashier;

    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Cashier::calculateTaxes();
    }

Как только расчет налога будет включен, все новые подписки и любые сгенерированные разовые счета-фактуры будут автоматически рассчитываться по налогу.

Чтобы эта функция работала должным образом, платежные данные вашего клиента, такие как имя, адрес и идентификационный номер налогоплательщика, должны быть синхронизированы с Stripe. Для этого вы можете использовать методы [синхронизации данных клиента] (#syncing-customer-data-with-stripe) и [Идентификатор налогоплательщика](#tax-ids), предлагаемые Cashier.

<a name="logging"></a>
### Логирование

Cashier позволяет вам указать канал регистрации, который будет использоваться при регистрации фатальных ошибок Stripe. Вы можете указать канал ведения журнала, определив переменную среды `CASHIER_LOGGER` в файле `.env` вашего приложения:

```ini
CASHIER_LOGGER=stack
```

Исключения, генерируемые вызовами API для Stripe, будут регистрироваться через канал журнала вашего приложения по умолчанию.

<a name="using-custom-models"></a>
### Использование пользовательских моделей

Вы можете свободно расширять модели, используемые внутри Cashier, определив свою собственную модель и расширив соответствующую модель Cashier:

    use Laravel\Cashier\Subscription as CashierSubscription;

    class Subscription extends CashierSubscription
    {
        // ...
    }

После определения вашей модели вы можете указать Cashier использовать вашу пользовательскую модель с помощью класса `Laravel\Cashier\Cashier`. Как правило, вы должны сообщить Cashier о ваших пользовательских моделях в методе `boot` класса `App\Providers\AppServiceProvider` вашего приложения:

    use App\Models\Cashier\Subscription;
    use App\Models\Cashier\SubscriptionItem;

    /**
     * Загрузка любых сервисов приложения.
     */
    public function boot(): void
    {
        Cashier::useSubscriptionModel(Subscription::class);
        Cashier::useSubscriptionItemModel(SubscriptionItem::class);
    }

<a name="quickstart"></a>
## Быстрый старт

<a name="quickstart-selling-products"></a>
### Продажа продуктов

> [!NOTE]
> Прежде чем использовать Stripe Checkout, вам следует определить продукты с фиксированными ценами в вашей панели управления Stripe. Кроме того, вы должны [настроить обработку веб-хуков Cashier](#handling-stripe-webhooks).

Предоставление продуктов и выставление счетов за подписки через ваше приложение может быть вызывающим трепет задачей. Однако благодаря Cashier и [Stripe Checkout](https://stripe.com/payments/checkout), вы легко можете создавать современные, надежные платежные интеграции.

Чтобы взимать оплату у клиентов за нерегулярные, одноразовые продукты, мы будем использовать Cashier для направления клиентов в Stripe Checkout, где они предоставят свои данные для оплаты и подтвердят свою покупку. После того, как оплата будет произведена через Checkout, клиент будет перенаправлен на URL успешного завершения, выбранный вами в вашем приложении:

    use Illuminate\Http\Request;

    Route::get('/checkout', function (Request $request) {
        $stripePriceId = 'price_deluxe_album';

        $quantity = 1;

        return $request->user()->checkout([$stripePriceId => $quantity], [
            'success_url' => route('checkout-success'),
            'cancel_url' => route('checkout-cancel'),
        ]);
    })->name('checkout');

    Route::view('/checkout/success', 'checkout.success')->name('checkout-success');
    Route::view('/checkout/cancel', 'checkout.cancel')->name('checkout-cancel');

Как видно в приведенном выше примере, мы будем использовать предоставленный Cashier метод `checkout` для перенаправления клиента в Stripe Checkout для заданного "идентификатора цены". При использовании Stripe "цены" относятся к [определенным ценам для конкретных продуктов](https://stripe.com/docs/products-prices/how-products-and-prices-work).

При необходимости метод `checkout` автоматически создаст клиента в Stripe и подключит эту запись клиента Stripe к соответствующему пользователю в базе данных вашего приложения. После завершения сеанса оформления заказа клиент будет перенаправлен на специальную страницу успеха или отмены, где вы сможете отобразить информационное сообщение клиенту.

<a name="providing-meta-data-to-stripe-checkout"></a>
#### Предоставление метаданных для Stripe Checkout

При продаже продуктов обычно ведется отслеживание завершенных заказов и приобретенных продуктов с помощью моделей `Cart` (корзина) и `Order` (заказ), определенных вашим собственным приложением. При перенаправлении клиентов в Stripe Checkout для завершения покупки вам может потребоваться предоставить существующий идентификатор заказа, чтобы вы могли связать завершенную покупку с соответствующим заказом при возвращении клиента обратно в ваше приложение.

Для этого вы можете предоставить массив `metadata` методу `checkout`. Допустим, что ожидающий `Order` (заказ) создается в нашем приложении, когда пользователь начинает процесс оформления заказа. Помните, что модели `Cart` (корзина) и `Order` (заказ) в этом примере являются иллюстративными и не предоставляются Cashier. Вы вольны реализовать эти концепции в соответствии с потребностями вашего собственного приложения:

    use App\Models\Cart;
    use App\Models\Order;
    use Illuminate\Http\Request;

    Route::get('/cart/{cart}/checkout', function (Request $request, Cart $cart) {
        $order = Order::create([
            'cart_id' => $cart->id,
            'price_ids' => $cart->price_ids,
            'status' => 'incomplete',
        ]);

        return $request->user()->checkout($order->price_ids, [
            'success_url' => route('checkout-success').'?session_id={CHECKOUT_SESSION_ID}',
            'cancel_url' => route('checkout-cancel'),
            'metadata' => ['order_id' => $order->id],
        ]);
    })->name('checkout');

Как видно из приведенного выше примера, когда пользователь начинает процесс оформления заказа, мы предоставляем все идентификаторы цен, связанные с корзиной / заказом, методу `checkout`. Конечно, ваше приложение отвечает за связывание этих элементов с "корзиной" или заказом, когда клиент добавляет их. Мы также предоставляем идентификатор заказа для сеанса оформления заказа Stripe через массив `metadata`. Наконец, мы добавляем переменную шаблона `CHECKOUT_SESSION_ID` к маршруту успешного оформления заказа. Когда Stripe перенаправляет клиентов обратно в ваше приложение, эта переменная шаблона автоматически заполняется идентификатором сеанса оформления заказа.

Теперь давайте создадим маршрут успешного оформления заказа. Это маршрут, на который пользователи будут перенаправлены после завершения покупки через Stripe Checkout. Внутри этого маршрута мы можем получить идентификатор сеанса оформления заказа Stripe и соответствующий экземпляр Stripe Checkout, чтобы получить доступ к предоставленным метаданным и обновить заказ вашего клиента соответственно:

    use App\Models\Order;
    use Illuminate\Http\Request;
    use Laravel\Cashier\Cashier;

    Route::get('/checkout/success', function (Request $request) {
        $sessionId = $request->get('session_id');

        if ($sessionId === null) {
            return;
        }

        $session = Cashier::stripe()->checkout->sessions->retrieve($sessionId);

        if ($session->payment_status !== 'paid') {
            return;
        }

        $orderId = $session['metadata']['order_id'] ?? null;

        $order = Order::findOrFail($orderId);

        $order->update(['status' => 'completed']);

        return view('checkout-success', ['order' => $order]);
    })->name('checkout-success');

Пожалуйста, обратитесь к документации Stripe для получения дополнительной информации о [данных, содержащихся в объекте сеанса оформления заказа](https://stripe.com/docs/api/checkout/sessions/object).

<a name="quickstart-selling-subscriptions"></a>
### Продажа подписок

> [!NOTE]
> Прежде чем использовать Stripe Checkout, вам следует определить продукты с фиксированными ценами в вашей панели управления Stripe. Кроме того, вы должны [настроить обработку веб-хуков Cashier](#handling-stripe-webhooks).

Предоставление продуктов и выставление счетов за подписки через ваше приложение может вызывать тревогу. Однако благодаря Cashier и [Stripe Checkout](https://stripe.com/payments/checkout), вы легко можете создавать современные, надежные платежные интеграции.

Чтобы узнать, как продавать подписки с использованием Cashier и Stripe Checkout, давайте рассмотрим простой сценарий подписки с базовым ежемесячным (`price_basic_monthly`) и годовым (`price_basic_yearly`) тарифами. Эти две цены могут быть объединены в продукт "Basic" (`pro_basic`) в нашей панели управления Stripe. Кроме того, наш сервис подписки может предлагать план Expert как `pro_expert`.

Сначала давайте узнаем, как клиент может подписаться на наши услуги. Конечно, можно предположить, что клиент нажмет кнопку "подписаться" базового плана на странице тарифов нашего приложения. Эта кнопка или ссылка должна направить пользователя на маршрут Laravel, который создает сеанс оформления заказа Stripe для выбранного им плана:

    use Illuminate\Http\Request;

    Route::get('/subscription-checkout', function (Request $request) {
        return $request->user()
            ->newSubscription('default', 'price_basic_monthly')
            ->trialDays(5)
            ->allowPromotionCodes()
            ->checkout([
                'success_url' => route('your-success-route'),
                'cancel_url' => route('your-cancel-route'),
            ]);
    });

Как видно из приведенного выше примера, мы перенаправляем клиента на сеанс оформления заказа Stripe, который позволит им подписаться на наш базовый план. После успешного оформления заказа или отмены клиент будет перенаправлен обратно на URL, который мы предоставили методу `checkout`. Чтобы узнать, когда их подписка фактически началась (поскольку некоторые способы оплаты требуют несколько секунд на обработку), нам также нужно [настроить обработку веб-хуков Cashier](#handling-stripe-webhooks).

Теперь, когда клиенты могут подписаться, нам нужно ограничить определенные части нашего приложения так, чтобы к ним имели доступ только подписанные пользователи. Конечно, мы всегда можем определить текущий статус подписки пользователя с помощью метода `subscribed`, предоставленного трейтом `Billable` Cashier:

```blade
@if ($user->subscribed())
    <p>Вы подписаны.</p>
@endif
```

Мы даже легко можем определить, подписан ли пользователь на конкретный продукт или цену:

```blade
@if ($user->subscribedToProduct('pro_basic'))
    <p>Вы подписаны на наш базовый продукт.</p>
@endif

@if ($user->subscribedToPrice('price_basic_monthly'))
    <p>Вы подписаны на наш ежемесячный базовый план.</p>
@endif
```

<a name="quickstart-building-a-subscribed-middleware"></a>
#### Создание middleware для подписанных пользователей

Для удобства вы можете создать [middleware](/docs/{{version}}/middleware), которое определяет, поступил ли входящий запрос от пользователя с подпиской. После того как это middleware будет определено, вы легко сможете назначить его маршруту, чтобы предотвратить доступ к маршруту пользователям без подписки:

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
                // Перенаправляем пользователя на страницу оплаты и просим его подписаться...
                return redirect('/billing');
            }

            return $next($request);
        }
    }

После определения middleware вы можете назначить его маршруту:

    use App\Http\Middleware\Subscribed;

    Route::get('/dashboard', function () {
        // ...
    })->middleware([Subscribed::class]);

<a name="quickstart-allowing-customers-to-manage-their-billing-plan"></a>
#### Разрешение клиентам управлять своим тарифным планом

Конечно, клиенты могут захотеть изменить свой тарифный план на другой продукт или "уровень". Самый простой способ это позволить - направить клиентов в [Портал выставления счетов для клиентов](https://stripe.com/docs/no-code/customer-portal) Stripe, который предоставляет пользовательский интерфейс, позволяющий клиентам загружать счета, обновлять свой способ оплаты и изменять тарифные планы.

Сначала определите ссылку или кнопку в вашем приложении, которая направляет пользователей на маршрут Laravel, используемый для инициации сеанса Портала выставления счетов:

```blade
<a href="{{ route('billing') }}">
    Billing
</a>
```

Далее определим маршрут, который инициирует сеанс Портала выставления счетов для клиента Stripe и перенаправляет пользователя в Портал. Метод `redirectToBillingPortal` принимает URL, на который пользователи должны вернуться при выходе из Портала:

    use Illuminate\Http\Request;

    Route::get('/billing', function (Request $request) {
        return $request->user()->redirectToBillingPortal(route('dashboard'));
    })->middleware(['auth'])->name('billing');

> [!NOTE]
> При условии, что вы настроили обработку веб-хуков Cashier, Cashier автоматически будет поддерживать таблицы вашего приложения, связанные с Cashier, в актуальном состоянии, осматривая входящие веб-хуки от Stripe. Так, например, когда пользователь отменяет свою подписку через Портал выставления счетов для клиента Stripe, Cashier получит соответствующий веб-хук и пометит подписку как "отмененную" в базе данных вашего приложения.

<a name="customers"></a>
## Клиенты

<a name="retrieving-customers"></a>
### Получение клиентов

Вы можете получить клиента по его идентификатору Stripe ID, используя метод `Cashier::findBillable`. Этот метод вернет экземпляр оплачиваемой модели:

    use Laravel\Cashier\Cashier;

    $user = Cashier::findBillable($stripeId);

<a name="creating-customers"></a>
### Создание клиентов

Иногда вы можете захотеть создать клиента Stripe, не начиная подписку. Вы можете выполнить это с помощью метода `createAsStripeCustomer`:

    $stripeCustomer = $user->createAsStripeCustomer();

После создания клиента в Stripe вы можете начать подписку в более поздний момент. Вы можете предоставить необязательный массив `$options`, чтобы передать любые дополнительные [параметры создания клиента, поддерживаемые Stripe API](https://stripe.com/docs/api/customers/create):

    $stripeCustomer = $user->createAsStripeCustomer($options);

Вы можете использовать метод `asStripeCustomer`, если хотите получить объект клиента Stripe для модели с возможностью выставления счетов:

    $stripeCustomer = $user->asStripeCustomer();

Метод `createOrGetStripeCustomer` может быть использован, если вы хотите получить объект клиента Stripe для заданной модели с возможностью выставления счетов, но не уверены, является ли модель с возможностью выставления счетов уже клиентом в Stripe. Этот метод создаст нового клиента в Stripe, если он еще не существует:

    $stripeCustomer = $user->createOrGetStripeCustomer();

<a name="updating-customers"></a>
### Обновление клиентов

Иногда вы можете захотеть обновить дополнительную информацию непосредственно на клиенте Stripe. Вы можете выполнить это с помощью метода `updateStripeCustomer`. Этот метод принимает массив [параметров обновления клиента, поддерживаемых Stripe API](https://stripe.com/docs/api/customers/update):

    $stripeCustomer = $user->updateStripeCustomer($options);

<a name="balances"></a>
### Балансы

Stripe позволяет увеличивать или уменьшать "баланс" клиента. Позже этот баланс будет учитываться при выставлении новых счетов. Чтобы проверить общий баланс клиента, вы можете использовать метод `balance`, доступный на вашей модели с возможностью выставления счетов.  Метод `balance` вернет форматированное строковое представление баланса в валюте клиента:

    $balance = $user->balance();

Чтобы пополнить баланс клиента, вы можете указать отрицательное значение для метода `applyBalance`. При желании вы также можете предоставить описание:

    $user->creditBalance(500, 'Premium customer top-up.');

Предоставление положительного значения методу `applyBalance` приведет к списанию средств с баланса клиента:

    $user->debitBalance(300, 'Bad usage penalty.');

Метод `applyBalance` создаст новые транзакции баланса клиента. Вы можете получить записи об этих транзакциях, используя метод `balanceTransactions`, что может быть полезно для предоставления журнала зачислений и списаний клиента для ознакомления:

    // Получить все транзакции...
    $transactions = $user->balanceTransactions();

    foreach ($transactions as $transaction) {
        // Сумма транзакции...
        $amount = $transaction->amount(); // $2.31

        // Получить связанный счет, если он доступен...
        $invoice = $transaction->invoice();
    }

<a name="tax-ids"></a>
### Идентификаторы налогоплательщиков

Cashier предлагает простой способ управления идетификаторами налогоплательщиков. Например, метод `taxIds` может быть использован для извлечения всех [идентификаторов налогоплательщиков](https://stripe.com/docs/api/customer_tax_ids/object), которые назначаются клиенту в качестве коллекции:

    $taxIds = $user->taxIds();

Вы также можете получить конкретный налоговый идентификатор клиента по его идентификатору:

    $taxId = $user->findTaxId('txi_belgium');

Вы можете создать новый налоговый идентификатор, указав действительный [тип](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-type) и значение для метода `createTaxId`:

    $taxId = $user->createTaxId('eu_vat', 'BE0123456789');

Метод `createTaxId` немедленно добавит идентификационный номер плательщика НДС в учетную запись клиента. [Проверка идентификаторов плательщика НДС также осуществляется Stripe](https://stripe.com/docs/invoicing/customer/tax-ids#validation); однако это асинхронный процесс. Вы можете получать уведомления об обновлениях проверки, подписавшись на веб-хук событие `customer.tax_id.updated` и проверив [параметр `verification` идентификаторов НДС](https://stripe.com/docs/api/customer_tax_ids/object#tax_id_object-verification). Для получения дополнительной информации об обработке веб-хуков, пожалуйста, обратитесь к [документации по определению обработчиков веб-хуков](#handling-stripe-webhooks).

Вы можете удалить налоговый идентификатор, используя метод `deleteTaxId`:

    $user->deleteTaxId('txi_belgium');

<a name="syncing-customer-data-with-stripe"></a>
### Синхронизация клиентских данных с помощью Stripe

Как правило, когда пользователи вашего приложения обновляют свое имя, адрес электронной почты или другую информацию, которая также хранится в Stripe, вы должны сообщить Stripe об обновлениях. Таким образом, ваша копия информации будет синхронизирована с копией вашего приложения.

Чтобы автоматизировать это, вы можете определить прослушиватель событий в вашей оплачиваемой модели, который реагирует на событие модели `updated`. Затем, в вашем прослушивателе событий, вы можете вызвать метод `syncStripeCustomerDetails` для модели:

    use App\Models\User;
    use function Illuminate\Events\queueable;

    /**
     * Метод модели "booted"(«Загруженный»).
     */
    protected static function booted(): void
    {
        static::updated(queueable(function (User $customer) {
            if ($customer->hasStripeId()) {
                $customer->syncStripeCustomerDetails();
            }
        }));
    }

Теперь каждый раз, когда обновляется ваша клиентская модель, ее информация будет синхронизироваться со Stripe. Для удобства Cashier автоматически синхронизирует информацию о вашем клиенте со Stripe при первоначальном создании клиента.

Вы можете настроить столбцы, используемые для синхронизации информации о клиентах со Stripe, переопределив различные методы, предоставляемые Cashier. Например, вы можете переопределить метод `stripeName`, чтобы настроить атрибут, который следует рассматривать как "имя" клиента, когда Cashier синхронизирует информацию о клиенте со Stripe:

    /**
     * Получите имя клиента, которое необходимо синхронизировать со Stripe.
     */
    public function stripeName(): string|null
    {
        return $this->company_name;
    }

Аналогичным образом, вы можете переопределить методы `stripeEmail`, `stripePhone` и `stripeAddress`. Эти методы будут синхронизировать информацию с соответствующими параметрами клиента при [обновлении объекта клиента Stripe](https://stripe.com/docs/api/customers/update). Если вы хотите получить полный контроль над процессом синхронизации информации о клиенте, вы можете переопределить метод `syncStripeCustomerDetails`.

<a name="billing-portal"></a>
### Биллинг портал

Stripe предлагает [простой способ настройки платежного портала](https://stripe.com/docs/billing/subscriptions/customer-portal), чтобы ваш клиент мог управлять своей подпиской, способами оплаты и просматривать историю выставления счетов. Вы можете перенаправить своих пользователей на портал выставления счетов, вызвав метод `redirectToBillingPortal` в модели выставления счетов с контроллера или маршрута:

    use Illuminate\Http\Request;

    Route::get('/billing-portal', function (Request $request) {
        return $request->user()->redirectToBillingPortal();
    });

По умолчанию, когда пользователь завершит управление своей подпиской, он сможет вернуться к маршруту `home` вашего приложения по ссылке на биллинговом портале Stripe. Вы можете указать URL, на который пользователь должен вернуться, передав его в качестве аргумента методу `redirectToBillingPortal`:

    use Illuminate\Http\Request;

    Route::get('/billing-portal', function (Request $request) {
        return $request->user()->redirectToBillingPortal(route('billing'));
    });

Если вы хотите сгенерировать URL к порталу выставления счетов без создания HTTP-ответа с перенаправлением, вы можете вызвать метод `billingPortalUrl`:

    $url = $request->user()->billingPortalUrl(route('billing'));

<a name="payment-methods"></a>
## Способы оплаты

<a name="storing-payment-methods"></a>
### Добавление способов оплаты

Чтобы создавать подписки или осуществлять "одноразовые" платежи с помощью Stripe, вам необходимо сохранить способ оплаты и получить его идентификатор из Stripe. Подход, используемый для достижения этой цели, отличается в зависимости от того, планируете ли вы использовать способ оплаты подписки или разовых платежей, поэтому ниже мы рассмотрим оба способа.

<a name="payment-methods-for-subscriptions"></a>
#### Способы оплаты для подписок

При сохранении информации о кредитной карте клиента для будущего использования по подписке необходимо использовать API Stripe "Setup Intents" для безопасного сбора информации о способе оплаты клиента. "Setup Intent" указывает Stripe на намерение взимать плату с способа оплаты клиента. Трейт `Billable` Cashier включает метод `createSetupIntent`, позволяющий легко создать новый Setup Intent. Вы должны вызвать этот метод из маршрута или контроллера, который отобразит форму, в которой будут собраны данные о способе оплаты вашего клиента:

    return view('update-payment-method', [
        'intent' => $user->createSetupIntent()
    ]);

После того, как вы создали Setup Intent и передали его в представление, вы должны прикрепить его секрет к элементу, который будет собирать информацию о способе оплаты. Например, рассмотрим эту форму "обновить способ оплаты":

```html
<input id="card-holder-name" type="text">

<!-- Заполнитель элементов Stripe -->
<div id="card-element"></div>

<button id="card-button" data-secret="{{ $intent->client_secret }}">
    Обновить способ оплаты
</button>
```

Затем можно использовать библиотеку Stripe.js для прикрепления [элемента Stripe](https://stripe.com/docs/stripe-js) к форме и безопасного сбора данных о платеже клиента:

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

Затем карта может быть верифицирована, и безопасный "идентификатор способа оплаты" может быть получен из Stripe с помощью [метода Stripe `confirmCardSetup`](https://stripe.com/docs/js/setup_intents/confirm_card_setup):

```js
const cardHolderName = document.getElementById('card-holder-name');
const cardButton = document.getElementById('card-button');
const clientSecret = cardButton.dataset.secret;

cardButton.addEventListener('click', async (e) => {
    const { setupIntent, error } = await stripe.confirmCardSetup(
        clientSecret, {
            payment_method: {
                card: cardElement,
                billing_details: { name: cardHolderName.value }
            }
        }
    );

    if (error) {
        // Отображение «error.message» пользователю...
    } else {
        // Карта успешно проверена...
    }
});
```

После того, как карта была верифицирована Stripe, вы можете передать полученный идентификатор `setupIntent.payment_method` в ваше приложение Laravel, где он может быть прикреплен к клиенту. Способ оплаты может быть либо [добавлен в качестве нового способа оплаты] (#adding-payment-methods), либо [использован для обновления способа оплаты по умолчанию] (#updating-the-default-payment-method). Вы также можете немедленно использовать идентификатор способа оплаты для [создания новой подписки] (#creating-subscriptions).

> [!NOTE]
> {tip} Если вы хотите получить дополнительную информацию о Setup Intents и сборе платежных реквизитов клиентов, пожалуйста, [ознакомьтесь с этим обзором, предоставленным Stripe](https://stripe.com/docs/payments/save-and-reuse#php).

<a name="payment-methods-for-single-charges"></a>
#### Способы оплаты для единовременных платежей

Конечно, при однократном списании средств с платежного метода клиента нам нужно будет использовать идентификатор платежного метода только один раз. Из-за ограничений Stripe вы не можете использовать сохраненный способ оплаты клиента по умолчанию для разовых платежей. Вы должны разрешить клиенту ввести данные о своем способе оплаты, используя библиотеку Stripe.js. Например, рассмотрим следующую форму:

```html
<input id="card-holder-name" type="text">

<!-- Заполнитель элементов Stripe -->
<div id="card-element"></div>

<button id="card-button">
    Обработка платежа
</button>
```

После определения такой формы, библиотека Stripe.js может быть использована для прикрепления [элемента Stripe](https://stripe.com/docs/stripe-js) в форму и надежно собирает платежные реквизиты клиента:

```html
<script src="https://js.stripe.com/v3/"></script>

<script>
    const stripe = Stripe('stripe-public-key');

    const elements = stripe.elements();
    const cardElement = elements.create('card');

    cardElement.mount('#card-element');
</script>
```

Затем карта может быть верифицирована, и безопасный "идентификатор способа оплаты" может быть получен из Stripe с помощью [метода Stripe `createPaymentMethod`](https://stripe.com/docs/stripe-js/reference#stripe-create-payment-method):

```js
const cardHolderName = document.getElementById('card-holder-name');
const cardButton = document.getElementById('card-button');

cardButton.addEventListener('click', async (e) => {
    const { paymentMethod, error } = await stripe.createPaymentMethod(
        'card', cardElement, {
            billing_details: { name: cardHolderName.value }
        }
    );

    if (error) {
        // Отображение «error.message» пользователю...
    } else {
        // Карта успешно проверена...
    }
});
```

Если верификация карты прошла успешно, вы можете передать `paymentMethod.id` вашему приложению Laravel и обработать [одноразовую оплату](#simple-charge).

<a name="retrieving-payment-methods"></a>
### Получение способов оплаты

Метод `PaymentMethod` в экземпляре оплачиваемой модели возвращает коллекцию экземпляров `Laravel\Cashier\PaymentMethod`:

    $paymentMethods = $user->paymentMethods();

По умолчанию этот метод возвращает способы оплаты типа `card`. Чтобы получить способы оплаты другого типа, вы можете передать `type` в качестве аргумента методу:

    $paymentMethods = $user->paymentMethods('sepa_debit');

Чтобы получить способ оплаты клиента по умолчанию, может быть использован метод `defaultPaymentMethod`.:

    $paymentMethod = $user->defaultPaymentMethod();

Вы можете получить конкретный способ оплаты, который привязан к оплачиваемой модели, используя метод `findPaymentMethod`:

    $paymentMethod = $user->findPaymentMethod($paymentMethodId);

<a name="payment-method-presence"></a>
### Наличие способа оплаты

Чтобы определить, имеет ли модель с возможностью выставления счетов привязанный к ее учетной записи способ оплаты по умолчанию, вызовите метод `hasDefaultPaymentMethod`:

    if ($user->hasDefaultPaymentMethod()) {
        // ...
    }

Вы можете использовать метод `hasPaymentMethod`, чтобы определить, имеет ли модель с возможностью выставления счетов хотя бы один способ оплаты, привязанный к ее учетной записи:

    if ($user->hasPaymentMethod()) {
        // ...
    }

Этот метод определит, имеет ли модель с возможностью выставления счетов хотя бы один способ оплаты. Чтобы определить, существует ли способ оплаты определенного типа для модели, вы можете передать `type` в качестве аргумента метода:

    if ($user->hasPaymentMethod('sepa_debit')) {
        // ...
    }

<a name="updating-the-default-payment-method"></a>
### Обновление способа оплаты по умолчанию

Метод `updateDefaultPaymentMethod` может использоваться для обновления информации о способе оплаты клиента по умолчанию. Этот метод принимает идентификатор платежного метода Stripe и назначает новый способ оплаты в качестве способа выставления счетов по умолчанию:

    $user->updateDefaultPaymentMethod($paymentMethod);

Чтобы синхронизировать информацию о вашем способе оплаты по умолчанию с информацией о способе оплаты клиента по умолчанию в Stripe, вы можете использовать метод `updateDefaultPaymentMethodFromStripe`:

    $user->updateDefaultPaymentMethodFromStripe();

> [!WARNING]
> Способ оплаты по умолчанию для клиента можно использовать только для выставления счетов и создания новых подписок. Из-за ограничений, налагаемых Stripe, его нельзя использовать для разовых платежей.

<a name="adding-payment-methods"></a>
### Добавление способа оплаты

Чтобы добавить новый способ оплаты, вы можете вызвать метод `addPaymentMethod` в оплачиваемой модели, передав идентификатор способа оплаты:

    $user->addPaymentMethod($paymentMethod);

> [!NOTE]
> Чтобы узнать, как получить идентификаторы способов оплаты, пожалуйста, ознакомьтесь с [документацией по хранению способов оплаты](#storing-payment-methods).

<a name="deleting-payment-methods"></a>
### Удаление способов оплаты

Чтобы удалить способ оплаты, вы можете вызвать метод `delete` в экземпляре `Laravel\Cashier\PaymentMethod`, который вы хотите удалить:

    $paymentMethod->delete();

Метод `deletePaymentMethod` удалит определенный способ оплаты из оплачиваемой модели:

    $user->deletePaymentMethod('pm_visa');

Метод `deletePaymentMethods` удалит всю информацию о способе оплаты для оплачиваемой модели:

    $user->deletePaymentMethods();

По умолчанию этот метод приведет к удалению способов оплаты типа `card`. Чтобы удалить способы оплаты другого типа, вы можете передать `type` в качестве аргумента методу:

    $user->deletePaymentMethods('sepa_debit');

> [!WARNING]
> Если у пользователя активная подписка, ваше приложение не должно позволять ему удалять способ оплаты по умолчанию.

<a name="subscriptions"></a>
## Подписки

Подписки предоставляют возможность настроить периодические платежи для ваших клиентов. Подписки Stripe, управляемые Cashier, обеспечивают поддержку нескольких тарифов подписки, количества подписок, пробных версий и многого другого.

<a name="creating-subscriptions"></a>
## Создание подписок

Чтобы создать подписку, сначала извлеките экземпляр вашей оплачиваемой модели, которая обычно будет экземпляром `App\Models\User`. После того, как вы извлекли экземпляр модели, вы можете использовать метод `newSubscription` для создания подписки на модель:

    use Illuminate\Http\Request;

    Route::post('/user/subscribe', function (Request $request) {
        $request->user()->newSubscription(
            'default', 'price_monthly'
        )->create($request->paymentMethodId);

        // ...
    });

Первым аргументом, передаваемым методу `newSubscription`, должно быть внутреннее имя подписки. Если ваше приложение предлагает только одну подписку, вы можете назвать ее `default` или `primary`. Это имя подписки предназначено только для внутреннего использования приложением и не предназначено для показа пользователям. Кроме того, он не должен содержать пробелов и никогда не должен быть изменен после создания подписки. Второй аргумент - это конкретный тариф, на который подписывается пользователь. Это значение должно соответствовать идентификатору тарифа в Stripe.

Метод `create`, который принимает [идентификатор способа оплаты Stripe](#storing-payment-methods) или объект Stripe `PaymentMethod`, запустит подписку, а также обновит вашу базу данных идентификатором клиента Stripe для оплачиваемой модели и другой соответствующей платежной информацией.

> [!WARNING]
> Передача идентификатора способа оплаты непосредственно в метод подписки `create` также автоматически добавит его в сохраненные способы оплаты пользователя.

<a name="collecting-recurring-payments-via-invoice-emails"></a>
#### Сбор периодических платежей через выставление счетов по электронной почте

Вместо автоматического сбора периодических платежей клиента вы можете поручить Stripe отправлять клиенту счет по электронной почте каждый раз, когда наступает срок оплаты. Затем клиент может вручную оплатить счет, как только он его получит. Клиенту не нужно заранее указывать способ оплаты при получении периодических платежей по счетам:

    $user->newSubscription('default', 'price_monthly')->createAndSendInvoice();

Время, в течение которого клиент должен оплатить свой счет до отмены подписки,  определяется параметром `days_until_due`. По умолчанию это 30 дней; однако вы можете указать конкретное значение для этого параметра, если хотите:

    $user->newSubscription('default', 'price_monthly')->createAndSendInvoice([], [
        'days_until_due' => 30
    ]);

<a name="subscription-quantities"></a>
#### «Количество» в подписках

Если вы хотите установить конкретное [количество](https://stripe.com/docs/billing/subscriptions/quantities) для получения цены при создании подписки вам следует вызвать метод `quantity` в конструкторе подписок перед созданием подписки:

    $user->newSubscription('default', 'price_monthly')
         ->quantity(5)
         ->create($paymentMethod);

<a name="additional-details"></a>
#### Дополнительные сведения

Если вы хотите указать дополнительные параметры [клиенту](https://stripe.com/docs/api/customers/create) или [подписке](https://stripe.com/docs/api/subscriptions/create), поддерживаемые Stripe, вы можете сделать это, передав их в качестве второго и третьего аргументов методу `create`:

    $user->newSubscription('default', 'price_monthly')->create($paymentMethod, [
        'email' => $email,
    ], [
        'metadata' => ['note' => 'Some extra information.'],
    ]);

<a name="coupons"></a>
#### Купоны

Если вы хотите применить купон при создании подписки, вы можете использовать метод `withCoupon`:

    $user->newSubscription('default', 'price_monthly')
         ->withCoupon('code')
         ->create($paymentMethod);

Или, если вы хотите применить [промокод Stripe](https://stripe.com/docs/billing/subscriptions/discounts/codes), вы можете использовать метод `withPromotionCode`:

    $user->newSubscription('default', 'price_monthly')
         ->withPromotionCode('promo_code_id')
         ->create($paymentMethod);

Указанный идентификатор промо-кода должен быть идентификатором Stripe API, присвоенным промо-коду, а не промо-кодом, с которым сталкивается клиент. Если вам нужно найти идентификатор промо-кода на основе предоставленного клиентского промо-кода, вы можете использовать метод `findPromotionCode`:

    // Найдите идентификатор промо-кода по его промо-коду клиента...
    $promotionCode = $user->findPromotionCode('SUMMERSALE');

    // Находим идентификатор активного промо-кода по его промо-коду клиента...
    $promotionCode = $user->findActivePromotionCode('SUMMERSALE');

В приведенном выше примере возвращаемый объект `$promotionCode` является экземпляром `Laravel\Cashier\PromotionCode`. Этот класс декорирует базовый объект `Stripe\PromotionCode`. Вы можете получить купон, связанный с промо-кодом, вызвав метод `coupon`:

    $coupon = $user->findPromotionCode('SUMMERSALE')->coupon();

Экземпляр купона позволяет определить сумму скидки и указать, представляет ли купон фиксированную скидку или скидку в процентах:

    if ($coupon->isPercentage()) {
        return $coupon->percentOff().'%'; // 21.5%
    } else {
        return $coupon->amountOff(); // $5.99
    }

Вы также можете получить скидки, которые в настоящее время применяются к клиенту или подписке:

    $discount = $billable->discount();

    $discount = $subscription->discount();

Возвращаемые экземпляры `Laravel\Cashier\Discount` декорируют базовый экземпляр объекта `Stripe\Discount`. Вы можете получить купон, связанный с этой скидкой, вызвав метод `coupon`:

    $coupon = $subscription->discount()->coupon();

Если вы хотите применить новый купон или промо-код к клиенту или подписке, вы можете сделать это с помощью методов `applyCoupon` или `applyPromotionCode`:

    $billable->applyCoupon('coupon_id');
    $billable->applyPromotionCode('promotion_code_id');

    $subscription->applyCoupon('coupon_id');
    $subscription->applyPromotionCode('promotion_code_id');

Помните, что вы должны использовать идентификатор API Stripe, назначенный промо-коду, а не клиентский промо-код. В любой момент времени к одному клиенту или подписке может быть применен только один купон или промо-код.

Для получения дополнительной информации по этому вопросу обратитесь к документации Stripe по [купонам](https://stripe.com/docs/billing/subscriptions/coupons) и [промо-кодам](https://stripe.com/docs/billing/subscriptions/coupons/codes).

<a name="adding-subscriptions"></a>
#### Добавление подписок

Если вы хотите добавить подписку клиенту, у которого уже есть способ оплаты по умолчанию, вы можете вызвать метод `add` в конструкторе подписок:

    use App\Models\User;

    $user = User::find(1);

    $user->newSubscription('default', 'price_monthly')->add();

<a name="creating-subscriptions-from-the-stripe-dashboard"></a>
#### Создание подписок с помощью панели управления Stripe

Вы также можете создавать подписки с помощью самой панели управления Stripe. При этом Cashier синхронизирует вновь добавленные подписки и присваивает им имя `default`. Чтобы настроить имя подписки, которое присваивается подпискам, созданным на панели управления, [расширьте `WebhookController`](#defining-webhook-event-handlers) и перезапишите метод `newSubscriptionName`.

Кроме того, вы можете создать только один тип подписки через панель управления Stripe. Если ваше приложение предлагает несколько подписок с разными именами, через панель мониторинга Stripe можно добавить только один тип подписки.

Наконец, вы всегда должны быть уверены, что добавляете только одну активную подписку на каждый тип подписки, предлагаемый вашим приложением. Если у клиента есть две подписки `default`, Cashier будет использовать только самую последнюю добавленную подписку, даже если обе будут синхронизированы с базой данных вашего приложения.

<a name="checking-subscription-status"></a>
### Проверка статуса подписки

Как только клиент зарегистрируется в вашем приложении, вы можете легко проверить статус его подписки, используя различные удобные методы. Во-первых, метод `subscribed` возвращает `true`, если у клиента активная подписка, даже если в настоящее время срок действия подписки истекает. Метод `subscribed` принимает имя подписки в качестве своего первого аргумента:

    if ($user->subscribed('default')) {
        // ...
    }

Метод `subscribed` также является отличным кандидатом для [посредника роута](/docs/{{version}}/middleware), позволяя вам фильтровать доступ к маршрутам и контроллерам на основе статуса подписки пользователя:

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
            if ($request->user() && ! $request->user()->subscribed('default')) {
                // This user is not a paying customer...
                return redirect('/billing');
            }

            return $next($request);
        }
    }

Если вы хотите определить, находится ли пользователь все еще в пределах своего пробного периода, вы можете использовать его метод `onTrial`. Этот метод может быть полезен для определения того, следует ли отображать предупреждение пользователю о том, что у него все еще действует пробный период:

    if ($user->subscription('default')->onTrial()) {
        // ...
    }

Метод `subscribedToProduct` может использоваться для определения того, подписан ли пользователь на данный продукт, на основе идентификатора данного продукта Stripe. В Stripe товары представляют собой наборы тарифов. В этом примере мы определим, является ли подписка пользователя `default` активной подпиской на "премиум" продукт приложения. Указанный идентификатор продукта Stripe должен соответствовать одному из идентификаторов вашего продукта на панели мониторинга Stripe:

    if ($user->subscribedToProduct('prod_premium', 'default')) {
        // ...
    }

Передавая массив методу `subscribedToProduct`, вы можете определить, является ли подписка пользователя `default` активной подпиской на "базовый" или "премиум" продукт приложения:

    if ($user->subscribedToProduct(['prod_basic', 'prod_premium'], 'default')) {
        // ...
    }

Метод `subscribedToPrice` может использоваться для определения того, соответствует ли подписка клиента заданному идентификатору тарифа:

    if ($user->subscribedToPrice('price_basic_monthly', 'default')) {
        // ...
    }

Метод `recurring` может быть использован для определения того, подписан ли пользователь в данный момент и не проходит ли пробный период:

    if ($user->subscription('default')->recurring()) {
        // ...
    }

> [!WARNING]
> Если у пользователя есть две подписки с одинаковым именем, самая последняя подписка всегда будет возвращена методом `subscription`. Например, у пользователя могут быть две записи подписки с именем `default`; однако одна из подписок может быть старой, срок действия которой истек, в то время как другая является текущей, активной подпиской. Самая последняя подписка всегда будет возвращена, в то время как более старые подписки хранятся в базе данных для просмотра истории.

<a name="cancelled-subscription-status"></a>
#### Canceled Subscription Status

Чтобы определить, был ли пользователь когда-то активным подписчиком, но отменил свою подписку, вы можете использовать метод `canceled`:

    if ($user->subscription('default')->canceled()) {
        // ...
    }

Вы также можете определить, отменил ли пользователь свою подписку, но все еще находится в "льготном периоде" до полного истечения срока действия подписки. Например, если пользователь отменяет подписку 5 марта, срок действия которой первоначально планировался на 10 марта, у пользователя действует "льготный период" до 10 марта. Обратите внимание, что метод `subscribed` все еще возвращает `true` в течение этого времени:

    if ($user->subscription('default')->onGracePeriod()) {
        // ...
    }

Чтобы определить, отменил ли пользователь свою подписку и больше не находится в пределах "льготного периода", вы можете использовать метод `ended`:

    if ($user->subscription('default')->ended()) {
        // ...
    }

<a name="incomplete-and-past-due-status"></a>
#### Статус незавершенного и просроченного платежа

Если подписка требует повторной оплаты после создания, подписка будет помечена как `incomplete`. Статусы подписок хранятся в столбце `stripe_status` таблицы `subscriptions` базы данных Cashier.

Аналогично, если при замене тарифов требуется вторичное платежное действие, подписка будет помечена как `past_due`. Если ваша подписка находится в любом из этих состояний, она не будет активна до тех пор, пока клиент не подтвердит свой платеж. Определение того, имеет ли подписка неполную оплату, может быть выполнено с использованием метода `hasIncompletePayment` в оплачиваемой модели или экземпляре подписки:

    if ($user->hasIncompletePayment('default')) {
        // ...
    }

    if ($user->subscription('default')->hasIncompletePayment()) {
        // ...
    }

Если подписка оплачена не полностью, вы должны направить пользователя на страницу подтверждения оплаты в Cashier, указав идентификатор `latestPayment`. Вы можете использовать метод `latestPayment`, доступный в экземпляре подписки, для получения этого идентификатора:

```html
<a href="{{ route('cashier.payment', $subscription->latestPayment()->id) }}">
    Please confirm your payment.
</a>
```

Если вы хотите, чтобы подписка по-прежнему считалась активной, когда она находится в состоянии `past_due` или `incomplete`, вы можете использовать методы `keepPastDueSubscriptionsActive` и `keepIncompleteSubscriptionsActive`, предоставленные Cashier. Обычно эти методы следует вызывать в методе `register` вашего `App\Providers\AppServiceProvider`:

    use Laravel\Cashier\Cashier;

    /**
     * Register any application services.
     */
    public function register(): void
    {
        Cashier::keepPastDueSubscriptionsActive();
        Cashier::keepIncompleteSubscriptionsActive();
    }

> [!WARNING]
> Когда подписка находится в состоянии `incomplete`, ее нельзя изменить до подтверждения платежа. Поэтому методы `swap` и `updateQuantity` вызовут исключение, когда подписка находится в состоянии `incomplete`.

<a name="subscription-scopes"></a>
#### Диапазоны подписки

Большинство состояний подписки также доступны в виде диапазона запросов, так что вы можете легко запрашивать в своей базе данных подписки, находящиеся в заданном состоянии:

    // Получить все активные подписки...
    $subscriptions = Subscription::query()->active()->get();

    // Получаем все отмененные подписки пользователя...
    $subscriptions = $user->subscriptions()->canceled()->get();

Полный список доступных диапазонов доступен ниже:

    Subscription::query()->active();
    Subscription::query()->canceled();
    Subscription::query()->ended();
    Subscription::query()->incomplete();
    Subscription::query()->notCanceled();
    Subscription::query()->notOnGracePeriod();
    Subscription::query()->notOnTrial();
    Subscription::query()->onGracePeriod();
    Subscription::query()->onTrial();
    Subscription::query()->pastDue();
    Subscription::query()->recurring();

<a name="changing-prices"></a>
### Изменение тарифов

После того, как клиент подписался на ваше приложение, он может иногда захотеть перейти на новый тариф в подписке. Чтобы перевести клиента на новый тариф, передайте идентификатор тарифа Stripe методу `swap`. При замене тарифов предполагается, что пользователь хотел бы повторно активировать свою подписку, если она была ранее отменена. Указанный идентификатор тарифа должен соответствовать идентификатору тарифа Stripe, доступному на панели управления Stripe:

    use App\Models\User;

    $user = App\Models\User::find(1);

    $user->subscription('default')->swap('price_yearly');

Если клиент находится на пробной версии, пробный период будет сохранен. Кроме того, если для подписки существует "количество", это количество также будет поддерживаться.

Если вы хотите поменять тарифы и отменить любой пробный период, на котором в данный момент находится клиент, вы можете воспользоваться методом `skipTrial`:

    $user->subscription('default')
            ->skipTrial()
            ->swap('price_yearly');

Если вы хотите поменять тарифы и немедленно выставить счет клиенту, не дожидаясь его следующего цикла выставления счетов, вы можете использовать метод `swapAndInvoice`:

    $user = User::find(1);

    $user->subscription('default')->swapAndInvoice('price_yearly');

<a name="prorations"></a>
#### Пропорции

По умолчанию Stripe пропорционально распределяет сборы при переключении между тарифами. Метод `noProrate` может быть использован для обновления тарифа подписки без пропорционального увеличения сборов:

    $user->subscription('default')->noProrate()->swap('price_yearly');

Для получения дополнительной информации о распределении подписок обратитесь к [документации Stripe](https://stripe.com/docs/billing/subscriptions/prorations).

> [!WARNING]
> Выполнение метода `noProrate` перед методом `swapAndInvoice` не окажет никакого влияния на распределение. Счет-фактура будет выставляться всегда.

<a name="subscription-quantity"></a>
### Количество подписок

Иногда подписки зависят от "количества". Например, приложение для управления проектами может взимать плату в размере $10 в месяц за каждый проект. Вы можете использовать методы `incrementQuantity` и `decrementQuantity` для удобного увеличения или уменьшения количества вашей подписки:

    use App\Models\User;

    $user = User::find(1);

    $user->subscription('default')->incrementQuantity();

    // Добавляем пять к текущему количеству подписки...
    $user->subscription('default')->incrementQuantity(5);

    $user->subscription('default')->decrementQuantity();

    // Вычитаем пять из текущего количества подписки...
    $user->subscription('default')->decrementQuantity(5);

В качестве альтернативы вы можете установить определенное количество, используя метод `updateQuantity`:

    $user->subscription('default')->updateQuantity(10);

Метод `noProrate` может быть использован для обновления количества подписок без пропорционального увеличения сборов:

    $user->subscription('default')->noProrate()->updateQuantity(10);

Для получения дополнительной информации о количестве подписок обратитесь к [документации Stripe](https://stripe.com/docs/subscriptions/quantities).

<a name="quantities-for-subscription-with-multiple-products"></a>
#### Количество подписок по разным ценам

Если ваша подписка является [многотарифной подпиской](#multiprice-subscriptions), вам следует передать название тарифа, количество которой вы хотите увеличить или уменьшить, в качестве второго аргумента методам increment / decrement:

    $user->subscription('default')->incrementQuantity(1, 'price_chat');

<a name="subscriptions-with-multiple-products"></a>
### Многотарифные подписки

[Многотарифные подписки](https://stripe.com/docs/billing/subscriptions/multiple-products) позволяют вам назначать несколько тарифов для выставления счетов одной подписке. Например, представьте, что вы создаете приложение "helpdesk" для обслуживания клиентов, базовая стоимость подписки на которое составляет 10 долларов в месяц, но которое предлагает дополнительный чат за дополнительные 15 долларов в месяц. Информация о подписке по нескольким тарифам хранится в таблице `subscription_items` базы данных Cashier.

Вы можете указать несколько тарифов для данной подписки, передав массив тарифов в качестве второго аргумента методу `newSubscription`:

    use Illuminate\Http\Request;

    Route::post('/user/subscribe', function (Request $request) {
        $request->user()->newSubscription('default', [
            'price_monthly',
            'price_chat',
        ])->create($request->paymentMethodId);

        // ...
    });

В приведенном выше примере к подписке клиента `default` будут привязаны два тарифа. Оба тарифа будут оплачиваться в соответствующих интервалах выставления счетов. При необходимости вы можете использовать метод `quantity`, чтобы указать конкретное количество для каждого тарифа:

    $user = User::find(1);

    $user->newSubscription('default', ['price_monthly', 'price_chat'])
        ->quantity(5, 'price_chat')
        ->create($paymentMethod);

Если вы хотите добавить другой тариф к существующей подписке, вы можете вызвать метод `addPrice` подписки:

    $user = User::find(1);

    $user->subscription('default')->addPrice('price_chat');

В приведенном выше примере будет добавлен новый тариф, и клиенту будет выставлен счет за него в следующем платежном цикле. Если вы хотите немедленно выставить счет клиенту, вы можете воспользоваться методом `addPriceAndInvoice`:

    $user->subscription('default')->addPriceAndInvoice('price_chat');

Если вы хотите добавить тариф с определенным количеством, вы можете передать количество в качестве второго аргумента методов `addPrice` или `addPriceAndInvoice`:

    $user = User::find(1);

    $user->subscription('default')->addPrice('price_chat', 5);

Вы можете удалить тарифы из подписок, используя метод `removePrice`:

    $user->subscription('default')->removePrice('price_chat');

> [!WARNING]
> Вы не имеете права отменять последний тариф в подписке. Вместо этого вам следует просто отменить подписку.

<a name="swapping-prices"></a>
#### Изменение тарифов

Вы также можете изменить тарифы, привязанные к мультитарифной подписке. Например, представьте, что у клиента есть подписка `price_basic` с дополнительным тарифом `price_chat`, и вы хотите обновить тариф клиента с `price_basic` до `price_pro`:

    use App\Models\User;

    $user = User::find(1);

    $user->subscription('default')->swap(['price_pro', 'price_chat']);

При выполнении приведенного выше примера базовый элемент подписки с `price_basic` удаляется, а элемент с `price_chat` сохраняется. Кроме того, создается новый элемент подписки для `price_pro`.

Вы также можете указать параметры элемента подписки, передав массив пар ключ / значение методу `swap`. Например, вам может потребоваться указать стоимость подписки:

    $user = User::find(1);

    $user->subscription('default')->swap([
        'price_pro' => ['quantity' => 5],
        'price_chat'
    ]);

Если вы хотите поменять один тариф на подписку, вы можете сделать это, используя метод `swap` для самого элемента подписки. Такой подход особенно полезен, если вы хотите сохранить все существующие метаданные о подписках и других тарифах:

    $user = User::find(1);

    $user->subscription('default')
            ->findItemOrFail('price_basic')
            ->swap('price_pro');

<a name="proration"></a>
#### Пропорция

По умолчанию Stripe пропорционально распределяет расходы при добавлении или удалении тарифов из мультитарифной подписки. Если вы хотите произвести корректировку тарифов без пропорциональности, вам следует привязать метод `noProrate` к вашей операции над тарифами:

    $user->subscription('default')->noProrate()->removePrice('price_chat');

<a name="swapping-quantities"></a>
#### Изменение «количества» в подписках

Если вы хотите обновить количество по тарифам отдельных подписок, вы можете сделать это с помощью [существующих методов определения количества](#subscription-quantity), передав название тарифа в качестве дополнительного аргумента методу:

    $user = User::find(1);

    $user->subscription('default')->incrementQuantity(5, 'price_chat');

    $user->subscription('default')->decrementQuantity(3, 'price_chat');

    $user->subscription('default')->updateQuantity(10, 'price_chat');

> [!WARNING]
> Когда подписка имеет несколько тарифов, атрибуты `stripe_price` и `quantity` в модели `Subscription` будут равны `null`. Чтобы получить доступ к отдельным атрибутам тарифа, вы должны использовать связь `items`, доступную в модели `Subscription`.

<a name="subscription-items"></a>
#### Элементы подписки

Когда подписка имеет несколько тарифов, в таблице `subscription_items` вашей базы данных будет храниться несколько "элементов" подписки. Вы можете получить к ним доступ через связь `items` в подписке:

    use App\Models\User;

    $user = User::find(1);

    $subscriptionItem = $user->subscription('default')->items->first();

    // Получаем цену и количество Stripe для определенного товара...
    $stripePrice = $subscriptionItem->stripe_price;
    $quantity = $subscriptionItem->quantity;

Вы также можете получить конкретный тариф, используя метод `findItemOrFail`

    $user = User::find(1);

    $subscriptionItem = $user->subscription('default')->findItemOrFail('price_chat');

<a name="multiple-subscriptions"></a>
### Несколько подписок

Stripe позволяет вашим клиентам иметь одновременно несколько подписок. Например, вы можете вести спортивный зал, который предлагает подписку на плавание и подписку на тяжелую атлетику, и каждая подписка может иметь разные цены. Конечно, клиенты должны иметь возможность подписаться на один или оба плана.

Когда ваше приложение создает подписки, вы можете указать тип подписки методу `newSubscription`. Тип может быть любой строкой, которая представляет собой тип подписки, которую пользователь инициирует:

    use Illuminate\Http\Request;

    Route::post('/swimming/subscribe', function (Request $request) {
        $request->user()->newSubscription('swimming')
            ->price('price_swimming_monthly')
            ->create($request->paymentMethodId);

        // ...
    });

В этом примере мы инициировали ежемесячную подписку на плавание для клиента. Однако в последующем он может захотеть перейти на ежегодную подписку. При изменении подписки клиента мы можем просто поменять тариф на подписке `swimming`:

    $user->subscription('swimming')->swap('price_swimming_yearly');

Конечно, вы также можете полностью отменить подписку:

    $user->subscription('swimming')->cancel();

<a name="usage-based-billing"></a>
### Биллинг на основе использования

[Биллинг на основе использования](https://stripe.com/docs/billing/subscriptions/metered-billing) позволяет взимать плату с клиентов в зависимости от использования ими продукта в течение платежного цикла. Например, вы можете взимать плату с клиентов в зависимости от количества текстовых сообщений или электронных писем, которые они отправляют в месяц.

Чтобы начать применять биллинг по использованию, вам сначала необходимо создать новый продукт на панели инструментов Stripe с [моделью биллинга на основе использования](https://docs.stripe.com/billing/subscriptions/usage-based/implementation-guide) и [счетчиом (meter)](https://docs.stripe.com/billing/subscriptions/usage-based/recording-usage#configure-meter). После создания счетчика сохраните связанное имя события и идентификатор счетчика, которые вам понадобятся для отчета и получения данных об использовании. Затем используйте метод `meteredPrice`, чтобы добавить идентификатор измеренной цены в подписку клиента:

    use Illuminate\Http\Request;

    Route::post('/user/subscribe', function (Request $request) {
        $request->user()->newSubscription('default')
            ->meteredPrice('price_metered')
            ->create($request->paymentMethodId);

        // ...
    });

Вы также можете запустить "расчетную" подписку через [Stripe Checkout](#checkout заказ):

    $checkout = Auth::user()
            ->newSubscription('default', [])
            ->meteredPrice('price_metered')
            ->checkout();

    return view('your-checkout-view', [
        'checkout' => $checkout,
    ]);

<a name="reporting-usage"></a>
#### Использование отчетов

По мере того, как клиент будет использовать приложение, вы будете сообщать Stripe об этом, чтобы можно было точно выставить ему точный счет. Чтобы сообщить об использовании измерямого события, вы можете использовать метод `reportMeterEvent` в вашей модели `Billable`:

    $user = User::find(1);

    $user->reportMeterEvent('emails-sent');

По умолчанию к расчетному периоду добавляется "количество использований", равное 1. В качестве альтернативы, вы можете указать определенную сумму "использования", чтобы добавить ее к использованию клиента за расчетный период:

    $user = User::find(1);

    $user->reportMeterEvent('emails-sent', quantity: 15);

Чтобы получить сводку событий клиента для счетчика, вы можете использовать метод `meterEventSummaries` экземпляра `Billable`:

    $user = User::find(1);

    $meterUsage = $user->meterEventSummaries($meterId);

    $meterUsage->first()->aggregated_value // 10

Дополнительную информацию о сводках событий счетчика можно найти в [документации по объекту сводки событий счетчика](https://docs.stripe.com/api/billing/meter-event_summary/object) Stripe.

Чтобы [перечислить все счетчики](https://docs.stripe.com/api/billing/meter/list), вы можете использовать метод `meters` экземпляра `Billable`:

    $user = User::find(1);

    $user->meters();

<a name="subscription-taxes"></a>
### Налоги подписки

> [!WARNING]
> Вместо расчета налоговых ставок вручную, вы можете [автоматически рассчитать налоги с помощью Stripe Tax](#tax-configuration)

Чтобы указать налоговые ставки, которые пользователь платит по подписке, вам следует реализовать метод `taxRates` в вашей оплачиваемой модели и вернуть массив, содержащий идентификаторы налоговых ставок Stripe. Вы можете определить эти налоговые ставки в [вашей информационной панели Stripe](https://dashboard.stripe.com/test/tax-rates):

    /**
     * Налоговые ставки, которые должны применяться к подпискам клиента.
     *
     * @return array<int, string>
     */
    public function taxRates(): array
    {
        return ['txr_id'];
    }

Метод `taxRates` позволяет вам применять налоговую ставку для каждого отдельного клиента, что может быть полезно для базы пользователей, охватывающей несколько стран и налоговых ставок.

Если вы предлагаете подписку по нескольким тарифам, вы можете определить разные налоговые ставки для каждого тарифа, внедрив метод `priceTaxRates` в вашей оплачиваемой модели:

    /**
     * Налоговые ставки, которые должны применяться к подпискам клиента.
     *
     * @return array<string, array<int, string>>
     */
    public function priceTaxRates(): array
    {
        return [
            'price_monthly' => ['txr_id'],
        ];
    }

> [!WARNING]
> Метод `taxRates` применяется только к оплате подписки. Если вы используете Cashier для осуществления разовых платежей, вам нужно будет вручную указать налоговую ставку на тот момент.

<a name="syncing-tax-rates"></a>
#### Синхронизация налоговых ставок

При изменении жестко закодированных идентификаторов налоговых ставок, возвращаемых методом `taxRates`, налоговые настройки для любых существующих подписок пользователя останутся прежними. Если вы хотите обновить значение налога для существующих подписок новыми значениями `taxRates`, вам следует вызвать метод `syncTaxRates` в экземпляре подписки пользователя:

    $user->subscription('default')->syncTaxRates();

Это также позволит синхронизировать любые налоговые ставки по элементам подписки с несколькими тарифами. Если ваше приложение предлагает многотарифные подписки, вам следует убедиться, что ваша оплачиваемая модель реализует метод `priceTaxRates` [обсуждался выше](#subscription-taxes).

<a name="tax-exemption"></a>
#### Освобождение от уплаты налогов

Cashier также предлагает методы `isNotTaxExempt`, `isTaxExempt` и `reverseChargeApplies`, чтобы определить, освобожден ли клиент от уплаты налогов. Эти методы будут вызывать Stripe API для определения статуса освобождения клиента от уплаты налогов:

    use App\Models\User;

    $user = User::find(1);

    $user->isTaxExempt();
    $user->isNotTaxExempt();
    $user->reverseChargeApplies();

> [!WARNING]
> Эти методы также доступны для любого объекта `Laravel\Cashier\Invoice`. Однако при вызове объекта `Invoice` методы будут определять статус исключения на момент создания счёта.

<a name="subscription-anchor-date"></a>
### Дата привязки подписки

По умолчанию привязкой платежного цикла является дата создания подписки или, если используется пробный период, дата окончания пробной версии. Если вы хотите изменить дату привязки счета, вы можете использовать метод `anchorBillingCycleOn`:

    use Illuminate\Http\Request;

    Route::post('/user/subscribe', function (Request $request) {
        $anchor = Carbon::parse('first day of next month');

        $request->user()->newSubscription('default', 'price_monthly')
                    ->anchorBillingCycleOn($anchor->startOfDay())
                    ->create($request->paymentMethodId);

        // ...
    });

Для получения дополнительной информации об управлении циклами выставления счетов по подписке обратитесь к [документации по циклу выставления счетов Stripe](https://stripe.com/docs/billing/subscriptions/billing-cycle)

<a name="cancelling-subscriptions"></a>
### Отмена подписки

Чтобы отменить подписку, вызовите метод `cancel` в подписке пользователя:

    $user->subscription('default')->cancel();

Когда подписка отменяется, Cashier автоматически установит столбец `ends_at` в вашей таблице `subscriptions` базы данных. Этот столбец используется, чтобы узнать, когда метод `subscribed` должен начать возвращать `false`.

Например, если клиент отменяет подписку 1 марта, но завершение подписки не планировалось до 5 марта, метод `subscribed` будет продолжать возвращать `true` до 5 марта. Это делается потому, что пользователю обычно разрешается продолжать использовать приложение до окончания платежного цикла.

Вы можете определить, отменил ли пользователь свою подписку, но все еще находится в "льготном периоде", используя метод `onGracePeriod`:

    if ($user->subscription('default')->onGracePeriod()) {
        // ...
    }

Если вы хотите немедленно отменить подписку, вызовите метод `cancelNow` в подписке пользователя:

    $user->subscription('default')->cancelNow();

Если вы хотите немедленно отменить подписку и выставить счет за любое оставшееся неучтенным дозированное использование или новые / ожидающие оплаты элементы счета-фактуры, вызовите метод `cancelNowAndInvoice` для подписки пользователя:

    $user->subscription('default')->cancelNowAndInvoice();

Вы также можете отменить подписку в определенный момент времени:

    $user->subscription('default')->cancelAt(
        now()->addDays(10)
    );

Наконец, перед удалением связанной модели пользователя всегда следует отменить его подписки:

    $user->subscription('default')->cancelNow();

    $user->delete();

<a name="resuming-subscriptions"></a>
### Возобновление подписок

Если клиент отменил свою подписку, и вы хотите возобновить ее, вы можете вызвать метод `resume` для подписки. Клиент все еще должен находиться в пределах своего "льготного периода", чтобы возобновить подписку:

    $user->subscription('default')->resume();

Если клиент отменяет подписку, а затем возобновляет ее до того, как срок действия подписки полностью истечет, счет клиенту не будет выставлен немедленно. Вместо этого их подписка будет повторно активирована, и им будет выставлен счет в первоначальном платежном цикле.

<a name="subscription-trials"></a>
## Пробные периоды

<a name="with-payment-method-up-front"></a>
### С указанием способа оплаты

Если вы хотите предложить своим клиентам пробные периоды, предварительно собирая информацию о способе оплаты, вам следует использовать метод `trialDays` при создании своих подписок:

    use Illuminate\Http\Request;

    Route::post('/user/subscribe', function (Request $request) {
        $request->user()->newSubscription('default', 'price_monthly')
                    ->trialDays(10)
                    ->create($request->paymentMethodId);

        // ...
    });

Этот метод установит дату окончания пробного периода в записи подписки в базе данных и проинструктирует Stripe не начинать выставление счетов клиенту до истечения этой даты. При использовании метода `trialDays` Cashier перезапишет любой пробный период по умолчанию, настроенный для тарифа в Stripe.

> [!WARNING]
> Если подписка клиента не будет отменена до даты окончания пробной версии, с него будет снята плата, как только истечет срок действия пробной версии, поэтому вам следует обязательно уведомить своих пользователей о дате окончания пробной версии.

Метод `trialUntil` позволяет вам предоставить экземпляр `DateTime`, который указывает, когда должен закончиться пробный период:

    use Carbon\Carbon;

    $user->newSubscription('default', 'price_monthly')
                ->trialUntil(Carbon::now()->addDays(10))
                ->create($paymentMethod);

Вы можете определить, находится ли пользователь в пределах своего пробного периода, используя либо метод `onTrial` экземпляра пользователя, либо метод `onTrial` экземпляра подписки. Два приведенных ниже примера эквивалентны:

    if ($user->onTrial('default')) {
        // ...
    }

    if ($user->subscription('default')->onTrial()) {
        // ...
    }

Вы можете использовать метод `endTrial`, чтобы немедленно завершить пробную версию подписки:

    $user->subscription('default')->endTrial();

Чтобы определить, истек ли срок действия существующего пробного периода, вы можете использовать метод `hasExpiredTrial`:

    if ($user->hasExpiredTrial('default')) {
        // ...
    }

    if ($user->subscription('default')->hasExpiredTrial()) {
        // ...
    }

<a name="defining-trial-days-in-stripe-cashier"></a>
#### Определение пробных дней в Stripe / Cashier

Вы можете указать, сколько пробных дней будет действовать ваш тариф, на панели управления Stripe или всегда передавать их явно с помощью Cashier. Если вы решите определить пробные дни вашего тарифа в Stripe, вы должны знать, что новые подписки, включая новые подписки для клиента, у которого была подписка в прошлом, всегда будут получать пробный период, если вы явно не вызовете метод `skipTrial()`.

<a name="without-payment-method-up-front"></a>
### Без указания способа оплаты

Если вы хотите предлагать пробные периоды без предварительного сбора информации о способе оплаты пользователя, вы можете установить в столбце `trial_ends_at` в записи пользователя желаемую дату окончания пробной версии. Обычно это делается во время регистрации пользователя:

    use App\Models\User;

    $user = User::create([
        // ...
        'trial_ends_at' => now()->addDays(10),
    ]);

> [!WARNING]
> Обязательно добавьте [приведение даты](/docs/{{version}}/eloquent-mutators##date-casting) для атрибута `trial_ends_at` в определении класса вашей оплачиваемой модели.

Cashier называет этот тип пробной версии "общей пробной версией", поскольку она не привязана ни к одной существующей подписке. Метод `onTrial` в экземпляре оплачиваемой модели вернет `true`, если текущая дата не превышает значения `trial_ends_at`:

    if ($user->onTrial()) {
        // У пользователя закончился пробный период...
    }

Как только вы будете готовы создать фактическую подписку для пользователя, вы можете использовать метод `newSubscription`, как обычно:

    $user = User::find(1);

    $user->newSubscription('default', 'price_monthly')->create($paymentMethod);

Чтобы получить дату окончания пробной версии пользователя, вы можете использовать метод `trialEndsAt`. Этот метод вернет экземпляр даты Carbon, если пользователь находится на пробной версии, или `null`, если это не так. Вы также можете передать необязательный параметр имени подписки, если хотите получить дату окончания пробной версии для конкретной подписки, отличной от подписки по умолчанию:

    if ($user->onTrial()) {
        $trialEndsAt = $user->trialEndsAt('main');
    }

Вы также можете использовать метод `onGenericTrial`, если хотите точно знать, что пользователь находится в пределах своего "общего" пробного периода и еще не создал фактическую подписку:

    if ($user->onGenericTrial()) {
        // У пользователя закончился «общий» пробный период...
    }

<a name="extending-trials"></a>
### Продление пробного периода

Метод `extendTrial` позволяет вам продлить пробный период подписки после того, как подписка была создана. Если срок действия пробной версии уже истек и клиенту уже выставлен счет за подписку, вы все равно можете предложить ему расширенную пробную версию. Время, потраченное в течение пробного периода, будет вычтено из следующего счета клиента:

    use App\Models\User;

    $subscription = User::find(1)->subscription('default');

    // Завершить пробную версию через 7 дней...
    $subscription->extendTrial(
        now()->addDays(7)
    );

    // Добавьте к пробной версии еще 5 дней...
    $subscription->extendTrial(
        $subscription->trial_ends_at->addDays(5)
    );

<a name="handling-stripe-webhooks"></a>
## Обработка Stripe веб-хуков

> [!NOTE]
> Вы можете использовать [интерфейс Stripe CLI](https://stripe.com/docs/stripe-clip), чтобы помочь протестировать веб-хуки во время локальной разработки.

Stripe может уведомлять ваше приложение о различных событиях с помощью веб-хуков. По умолчанию маршрут, который указывает на контроллер веб-хука Cashier, автоматически регистрируется сервис-провайдером Cashier. Этот контроллер будет обрабатывать все входящие запросы веб-хука.

По умолчанию контроллер веб-хук Cashier автоматически обрабатывает отмену подписок, в которых слишком много сброшенных платежей (как определено в настройках Stripe), обновления клиентов, удаления клиентов, обновления подписки и изменения способа оплаты; однако, как мы скоро узнаем, вы можете расширить этот контроллер для обработки любого события вэб-хука Stripe как вам нравится.

Чтобы убедиться, что ваше приложение может обрабатывать веб-хуки Stripe, обязательно настройте URL-адрес веб-хука на панели управления Stripe. По умолчанию контроллер веб-хука Cashier отвечает на URL-адрес `/stripe/webhook`. Полный список всех веб-подключений, которые вы должны включить на панели управления Stripe, выглядит следующим образом:

- `customer.subscription.created`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `customer.updated`
- `customer.deleted`
- `payment_method.automatically_updated`
- `invoice.payment_action_required`
- `invoice.payment_succeeded`

Для удобства в Cashier включена команда Artisan `cashier:webhook`. Эта команда создаст веб-хука в Stripe, который прослушивает все события, требуемые Cashier:

```shell
php artisan cashier:webhook
```

По умолчанию созданный веб-хук будет указывать на URL, определенный переменной окружения `APP_URL` и `cashier.webhook` маршрут, который входит в комплект поставки Cashier. Вы можете указать параметр `--url` при вызове команды, если хотите использовать другой URL:

```shell
php artisan cashier:webhook --url "https://example.com/stripe/webhook"
```

Созданный веб-хук будет использовать версию Stripe API, с которой совместима ваша версия Cashier. Если вы хотите использовать другую версию Stripe, вы можете указать опцию `--api-version`:

```shell
php artisan cashier:webhook --api-version="2019-12-03"
```

После создания веб-хук будет немедленно активен. Если вы хотите создать веб-хук, но отключить его до тех пор, пока не будете готовы, вы можете указать опцию `--disabled` при вызове команды:

```shell
php artisan cashier:webhook --disabled
```

> [!WARNING]
> Убедитесь, что вы защищаете входящие запросы веб-хук Stripe с помощью встроенного промежуточного программного обеспечения Cashier [проверка подписи веб-хука](#verifying-webhook-signatures).

<a name="webhooks-csrf-protection"></a>
#### Веб-хуки и защита от CSRF

Поскольку веб-хуки Stripe необходимо обходить [защиту CSRF](/docs/{{version}}/csrf) Laravel, вам следует убедиться, что Laravel не пытается проверить токен CSRF для входящих веб-хуков Stripe. Для этого вам следует исключить `stripe/*` из защиты CSRF в файле `bootstrap/app.php` вашего приложения:

    ->withMiddleware(function (Middleware $middleware) {
        $middleware->validateCsrfTokens(except: [
            'stripe/*',
        ]);
    })

<a name="defining-webhook-event-handlers"></a>
### Определение веб-хука событий

Cashier автоматически обрабатывает отмены подписки в случае несвоевременных платежей и других распространенных событий веб-хука Stripe. Однако, если у вас есть дополнительные события вэб-хука, которые вы хотели бы обработать, вы можете сделать это, прослушав следующие события, отправляемые Cashier:

- `Laravel\Cashier\Events\WebhookReceived`
- `Laravel\Cashier\Events\WebhookHandled`

Оба события содержат полную полезную нагрузку веб-хука Stripe. Например, если вы хотите обработать веб-запрос `invoice.payment_succeeded`, вы можете зарегистрировать [прослушивателя](/docs/{{version}}/events#defining-listeners), который будет обрабатывать событие:

    <?php

    namespace App\Listeners;

    use Laravel\Cashier\Events\WebhookReceived;

    class StripeEventListener
    {
        /**
         * Обработка полученных веб-хуков Stripe.
         */
        public function handle(WebhookReceived $event): void
        {
            if ($event->payload['type'] === 'invoice.payment_succeeded') {
                // Обработка входящего события...
            }
        }
    }

<a name="verifying-webhook-signatures"></a>
### Проверка подписей веб-хука

Чтобы обезопасить свои веб-хуки, вы можете использовать [подписи Stripe веб-хука](https://stripe.com/docs/webhooks/signatures). Для удобства Cashier автоматически включает промежуточное программное обеспечение, которое проверяет правильность входящего запроса веб-хука Stripe.

Чтобы включить проверку веб-хука, убедитесь, что переменная окружения `STRIPE_WEBHOOK_SECRET` установлена в файле `.env` вашего приложения. `Secret` веб-хука можно получить с панели управления вашей учетной записи Stripe.

<a name="single-charges"></a>
## Разовые списания

<a name="simple-charge"></a>
### Разовое списание

Если вы хотите произвести единовременное списание средств с клиента, вы можете использовать метод `charge` для экземпляра модели, подлежащего оплате. Вам нужно будет [указать идентификатор способа оплаты](#payment-methods-for-single-charges) в качестве второго аргумента метода `charge`:

    use Illuminate\Http\Request;

    Route::post('/purchase', function (Request $request) {
        $stripeCharge = $request->user()->charge(
            100, $request->paymentMethodId
        );

        // ...
    });

Метод `charge` принимает массив в качестве своего третьего аргумента, позволяя вам передавать любые параметры, которые вы пожелаете, для базового процесса создания Stripe charge. Более подробную информацию о вариантах, доступных вам при создании платежей, можно найти в [документации Stripe](https://stripe.com/docs/api/charges/create):

    $user->charge(100, $paymentMethod, [
        'custom_option' => $value,
    ]);

Вы также можете использовать метод `charge` без участия основного клиента или пользователя. Чтобы выполнить это, вызовите метод `charge` в новом экземпляре оплачиваемой модели вашего приложения:

    use App\Models\User;

    $stripeCharge = (new User)->charge(100, $paymentMethod);

Метод `charge` выдаст исключение, если списание завершится неудачей. Если списание пройдет успешно, экземпляр `Laravel\Cashier\Payment` будет возвращен из метода:

    try {
        $payment = $user->charge(100, $paymentMethod);
    } catch (Exception $e) {
        // ...
    }

> [!WARNING]
> Метод `charge` принимает сумму, которую вы хотели бы списать, в наименьшем знаменателе валюты, используемой вашим приложением. Например, при использовании долларов США суммы следует указывать в пенни.

<a name="charge-with-invoice"></a>
### Списание со счетом

Иногда вам может потребоваться произвести единовременную оплату и предложить своему клиенту квитанцию в формате PDF. Метод `invoicePrice` позволяет вам сделать именно это. Например, давайте выставим клиенту счет за пять новых рубашек:

    $user->invoicePrice('price_tshirt', 5);

Счет будет немедленно списан с использованием способа оплаты, используемого пользователем по умолчанию. Метод `invoicePrice` также принимает массив в качестве своего третьего аргумента. Этот массив содержит параметры выставления счетов для элемента счета-фактуры. Четвертый аргумент, принимаемый методом, также является массивом, который должен содержать параметры выставления счета для самого счета-фактуры:

    $user->invoicePrice('price_tshirt', 5, [
        'discounts' => [
            ['coupon' => 'SUMMER21SALE']
        ],
    ], [
        'default_tax_rates' => ['txr_id'],
    ]);

Аналогично `invoicePrice`, вы можете использовать метод `tabPrice` для создания одноразового счета за несколько товаров (до 250 товаров на счете), добавив их в "вкладку" клиента, а затем выставив счет клиенту. Например, мы можем выставить счет клиенту за пять рубашек и две кружки:

    $user->tabPrice('price_tshirt', 5);
    $user->tabPrice('price_mug', 2);
    $user->invoice();

В качестве альтернативы, вы можете использовать метод `invoiceFor`, чтобы произвести "единовременную" оплату за счет способа оплаты, используемого клиентом по умолчанию:

    $user->invoiceFor('One Time Fee', 500);

Хотя вам доступен метод `invoiceFor`, рекомендуется использовать метод `invoicePrice` с заранее определенными ценами. Поступая таким образом, вы получите доступ к улучшенной аналитике и данным на панели мониторинга Stripe, касающимся ваших продаж по каждому продукту.

> [!WARNING]
> Методы `invoicePrice` и `invoiceFor` создадут накладную Stripe, которая повторит неудачные попытки выставления счета. Если вы не хотите, чтобы счета-фактуры повторяли неудачные начисления, вам нужно будет закрыть их с помощью Stripe API после первого неудачного начисления.

<a name="creating-payment-intents"></a>
### Создание платежных намерений

Вы можете создать новое платежное намерение Stripe, вызвав метод `pay` на экземпляре модели, подлежащей оплате. Вызов этого метода создаст платежное намерение, обернутое в экземпляр `Laravel\Cashier\Payment`:

    use Illuminate\Http\Request;

    Route::post('/pay', function (Request $request) {
        $payment = $request->user()->pay(
            $request->get('amount')
        );

        return $payment->client_secret;
    });

После создания платежного намерения вы можете вернуть клиентский секрет на фронтенд вашего приложения, чтобы пользователь мог завершить оплату в своем браузере. Чтобы узнать больше о создании полных платежных процессов с использованием платежных намерений Stripe, обратитесь к [документации Stripe](https://stripe.com/docs/payments/accept-a-payment?platform=web).

При использовании метода `pay` доступны по умолчанию те методы оплаты, которые включены в вашей панели управления Stripe. В качестве альтернативы, если вы хотите разрешить использование только определенных методов оплаты, вы можете использовать метод `payWith`:

    use Illuminate\Http\Request;

    Route::post('/pay', function (Request $request) {
        $payment = $request->user()->payWith(
            $request->get('amount'), ['card', 'bancontact']
        );

        return $payment->client_secret;
    });

> [!WARNING]
> Методы `pay` и `payWith` принимают сумму платежа в наименьшем знаменателе валюты, используемой вашим приложением. Например, если клиенты платят в долларах США, суммы должны быть указаны в центах.

<a name="refunding-charges"></a>
### Возврат списаниий

Если вам необходимо возместить стоимость Stripe, вы можете воспользоваться методом `refund`. Этот метод принимает Stripe [идентификатор намерения платежа](#payment-methods-for-single-charges) в качестве своего первого аргумента:

    $payment = $user->charge(100, $paymentMethodId);

    $user->refund($payment->id);

<a name="invoices"></a>
## Счета

<a name="retrieving-invoices"></a>
### Получение счетов

Вы можете легко получить массив счетов оплачиваемой модели, используя метод `invoices`. Метод `invoices` возвращает коллекцию экземпляров `Laravel\Cashier\Invoice`:

    $invoices = $user->invoices();

Если вы хотите включить в результаты отложенные счета вы можете использовать метод `invoicesIncludingPending`:

    $invoices = $user->invoicesIncludingPending();

Вы можете использовать метод `findInvoice` для получения конкретного счёта по его идентификатору:

    $invoice = $user->findInvoice($invoiceId);

<a name="displaying-invoice-information"></a>
#### Отображение информации о счете

При перечислении счетов для клиента вы можете использовать методы счёта для отображения соответствующей информации о счёте. Например, вы можете захотеть отобразить каждый счёт в таблице, что позволит пользователю легко загрузить любой из них:

```blade
<table>
    @foreach ($invoices as $invoice)
        <tr>
            <td>{{ $invoice->date()->toFormattedDateString() }}</td>
            <td>{{ $invoice->total() }}</td>
            <td><a href="/user/invoice/{{ $invoice->id }}">Download</a></td>
        </tr>
    @endforeach
</table>
```

<a name="upcoming-invoices"></a>
### Предстоящие счета

Чтобы получить предстоящий счет для клиента, вы можете использовать метод `upcomingInvoice`:

    $invoice = $user->upcomingInvoice();

Аналогично, если у клиента несколько подписок, вы также можете получить предстоящий счет-фактуру для конкретной подписки:

    $invoice = $user->subscription('default')->upcomingInvoice();

<a name="previewing-subscription-invoices"></a>
### Предварительный просмотр счетов-фактур по подписке

Используя метод `previewInvoice`, вы можете просмотреть счет-фактуру перед внесением изменений в тариф. Это позволит вам определить, как будет выглядеть счет вашего клиента при изменении тарифа:

    $invoice = $user->subscription('default')->previewInvoice('price_yearly');

Вы можете передать массив тарифов методу `previewInvoice`, чтобы просмотреть счета-фактуры с несколькими новыми тарифами:

    $invoice = $user->subscription('default')->previewInvoice(['price_yearly', 'price_metered']);

<a name="generating-invoice-pdfs"></a>
### Генерация счетов PDF

Перед тем как создавать PDF-файлы счетов, вы должны использовать Composer для установки библиотеки Dompdf, которая является рендерером счетов по умолчанию для Cashier:

```php
composer require dompdf/dompdf
```

Находясь внутри маршрута или контроллера, вы можете использовать метод `downloadInvoice` для создания PDF-загрузки данного счета-фактуры. Этот метод автоматически сгенерирует соответствующий HTTP-ответ, необходимый для загрузки счета-фактуры:

    use Illuminate\Http\Request;

    Route::get('/user/invoice/{invoice}', function (Request $request, string $invoiceId) {
        return $request->user()->downloadInvoice($invoiceId);
    });

По умолчанию все данные в счете-фактуре получены из данных клиента и счета-фактуры, хранящихся в Stripe. Однако вы можете настроить некоторые из этих данных, предоставив массив в качестве второго аргумента методу `downloadInvoice`. Этот массив позволяет вам настраивать информацию, такую как сведения о вашей компании и продукте:

    return $request->user()->downloadInvoice($invoiceId, [
        'vendor' => 'Ваша компания',
        'product' => 'Ваш продукт',
        'street' => 'Главная ул. 1',
        'location' => '2000 Антверпен, Бельгия',
        'phone' => '+32 499 00 00 00',
        'email' => 'info@example.com',
        'url' => 'https://example.com',
        'vendorVat' => 'BE123456789',
    ]);

Метод `downloadInvoice` также допускает пользовательское имя файла с помощью своего третьего аргумента. К этому имени файла автоматически будет добавлен суффикс `.pdf`:

    return $request->user()->downloadInvoice($invoiceId, [], 'my-invoice');

<a name="custom-invoice-render"></a>
#### Средство отображения пользовательских счетов

Cashier также позволяет использовать пользовательский инструмент отображения счетов-фактур. По умолчанию Cashier использует реализацию `DompdfInvoiceRenderer`, которая использует библиотеку PHP [dompdf](https://github.com/dompdf/dompdf) для генерации счетов Cashier. Однако вы можете использовать любой рендерер, который пожелаете, реализовав интерфейс `Laravel\Cashier\Contracts\InvoiceRenderer`. Например, вы можете захотеть отобразить PDF-файл счета-фактуры с помощью вызова API стороннего сервиса отображения PDF-файлов:

    use Illuminate\Support\Facades\Http;
    use Laravel\Cashier\Contracts\InvoiceRenderer;
    use Laravel\Cashier\Invoice;

    class ApiInvoiceRenderer implements InvoiceRenderer
    {
        /**
         * Отображение данного счета и возврат необработанных байтов PDF.
         */
        public function render(Invoice $invoice, array $data = [], array $options = []): string
        {
            $html = $invoice->view($data)->render();

            return Http::get('https://example.com/html-to-pdf', ['html' => $html])->get()->body();
        }
    }

После того, как вы внедрили контракт с обработчиком счетов-фактур, вам следует обновить значение конфигурации `cashier.invoices.renderer` в настройках файл конфигурации `config/cashier.php` вашего приложения. Это значение конфигурации должно быть установлено в качестве имени класса вашей пользовательской реализации средства визуализации.

<a name="checkout"></a>
## Оформление

Cashier Stripe также предоставляет поддержку [Stripe Checkout](https://stripe.com/payments/checkout). Stripe Checkout избавляет от необходимости внедрять пользовательские страницы для приема платежей, предоставляя предварительно созданную размещенную платежную страницу.

Следующая документация содержит информацию о том, как начать использовать Stripe Checkout с помощью Cashier. Чтобы узнать больше о Stripe Checkout, вам также следует рассмотреть возможность ознакомления с [собственной документацией Stripes по оформлению заказа](https://stripe.com/docs/payments/checkout).

<a name="product-checkouts"></a>
### Оформление заказа продукта

Вы можете выполнить оформление заказа для существующего продукта, который был создан в вашей информационной панели Stripe, используя метод `checkout` для оплачиваемой модели. Метод `checkout` инициирует новый сеанс оформления заказа Stripe. По умолчанию от вас требуется ввести идентификатор тарифа Stripe:

    use Illuminate\Http\Request;

    Route::get('/product-checkout', function (Request $request) {
        return $request->user()->checkout('price_tshirt');
    });

При необходимости вы также можете указать количество продукта:

    use Illuminate\Http\Request;

    Route::get('/product-checkout', function (Request $request) {
        return $request->user()->checkout(['price_tshirt' => 15]);
    });

Когда клиент посещает этот маршрут, он будет перенаправлен на страницу оформления заказа Stripe. По умолчанию, когда пользователь успешно завершает или отменяет покупку, он будет перенаправлен на ваш маршрут `home`, но вы можете указать пользовательские URL-адреса обратного вызова, используя опции `success_url` и `cancel_url`:

    use Illuminate\Http\Request;

    Route::get('/product-checkout', function (Request $request) {
        return $request->user()->checkout(['price_tshirt' => 1], [
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
    });

При определении вашей опции оформления заказа `success_url` вы можете указать Stripe добавить идентификатор сеанса оформления заказа в качестве параметра строки запроса при вызове вашего URL. Для этого добавьте буквальную строку `{CHECKOUT_SESSION_ID}` в строку вашего запроса `success_url`. Stripe заменит этот заполнитель фактическим идентификатором сеанса оформления заказа:

    use Illuminate\Http\Request;
    use Stripe\Checkout\Session;
    use Stripe\Customer;

    Route::get('/product-checkout', function (Request $request) {
        return $request->user()->checkout(['price_tshirt' => 1], [
            'success_url' => route('checkout-success').'?session_id={CHECKOUT_SESSION_ID}',
            'cancel_url' => route('checkout-cancel'),
        ]);
    });

    Route::get('/checkout-success', function (Request $request) {
        $checkoutSession = $request->user()->stripe()->checkout->sessions->retrieve($request->get('session_id'));

        return view('checkout.success', ['checkoutSession' => $checkoutSession]);
    })->name('checkout-success');

<a name="checkout-promotion-codes"></a>
#### Промокоды

По умолчанию Stripe Checkout не разрешает [промокоды, которые могут быть использованы пользователем](https://stripe.com/docs/billing/subscriptions/discounts/codes). К счастью, есть простой способ включить их на вашей странице оформления заказа. Для этого вы можете вызвать метод `allowPromotionCodes`:

    use Illuminate\Http\Request;

    Route::get('/product-checkout', function (Request $request) {
        return $request->user()
            ->allowPromotionCodes()
            ->checkout('price_tshirt');
    });

<a name="single-charge-checkouts"></a>
### Оформление одиночного списания

Вы также можете выполнить простую оплату за специальный продукт, который не был создан в вашей информационной панели Stripe. Для этого вы можете использовать метод `checkoutCharge` для модели, подлежащей оплате, и передать ей подлежащую оплате сумму, название продукта и необязательное количество. Когда клиент посещает этот маршрут, он будет перенаправлен на страницу оформления заказа Stripe:

    use Illuminate\Http\Request;

    Route::get('/charge-checkout', function (Request $request) {
        return $request->user()->checkoutCharge(1200, 'T-Shirt', 5);
    });

> [!WARNING]
> При использовании метода `checkoutCharge` Stripe всегда будет создавать новый продукт и тариф на вашей информационной панели Stripe. Поэтому мы рекомендуем вам предварительно создать товары на панели управления Stripe и вместо этого использовать метод `checkout`.

<a name="subscription-checkouts"></a>
### Оформление заказа подписки

> [!WARNING]
> Для использования Stripe Checkout для подписок требуется включить веб-хук `customer.subscription.created` на панели управления Stripe. Этот веб-хук создаст запись подписки в вашей базе данных и сохранит все соответствующие элементы подписки.

Вы также можете использовать Stripe Checkout для инициирования подписки. После определения вашей подписки с помощью методов построения подписки Cashier, вы можете вызвать метод `checkout`. Когда клиент посещает этот маршрут, он будет перенаправлен на страницу оформления заказа Stripe:

    use Illuminate\Http\Request;

    Route::get('/subscription-checkout', function (Request $request) {
        return $request->user()
            ->newSubscription('default', 'price_monthly')
            ->checkout();
    });

Как и в случае с проверкой товара, вы можете настроить URL-адреса для подтверждения и отмены заказа:

    use Illuminate\Http\Request;

    Route::get('/subscription-checkout', function (Request $request) {
        return $request->user()
            ->newSubscription('default', 'price_monthly')
            ->checkout([
                'success_url' => route('your-success-route'),
                'cancel_url' => route('your-cancel-route'),
            ]);
    });

Конечно, вы также можете включить промо-коды для оформления подписки:

    use Illuminate\Http\Request;

    Route::get('/subscription-checkout', function (Request $request) {
        return $request->user()
            ->newSubscription('default', 'price_monthly')
            ->allowPromotionCodes()
            ->checkout();
    });

> [!WARNING]
> К сожалению, Stripe Checkout не поддерживает все параметры выставления счетов при запуске подписки. Использование метода `anchorBillingCycleOn` в конструкторе подписок, настройка пропорционального поведения или настройка режима оплаты не будут иметь никакого эффекта во время сеансов оформления заказа Stripe. Пожалуйста, ознакомьтесь с [документацией Stripe Checkout Session API](https://stripe.com/docs/api/checkout/sessions/create), чтобы просмотреть, какие параметры доступны.

<a name="stripe-checkout-trial-periods"></a>
#### Оформление заказа Stripe и пробные периоды

Конечно, вы можете определить пробный период при создании подписки, которая будет завершена с помощью Stripe Checkout:

    $checkout = Auth::user()->newSubscription('default', 'price_monthly')
        ->trialDays(3)
        ->checkout();

Однако пробный период должен составлять не менее 48 часов, что является минимальным сроком пробной версии, поддерживаемым Stripe Checkout.

<a name="stripe-checkout-subscriptions-and-webhooks"></a>
#### Подписки и веб-хуки

Помните, что Stripe и Cashier обновляют статусы подписки через веб-хуки, поэтому существует вероятность того, что подписка может быть еще не активна, когда клиент вернется в приложение после ввода своей платежной информации. Чтобы справиться с этим сценарием, вы можете захотеть отобразить сообщение, информирующее пользователя о том, что его оплата или подписка ожидаются.

<a name="collecting-tax-ids"></a>
### Сбор идентификаторов налогоплательщиков

Оформление заказа также поддерживает сбор налогового идентификатора клиента. Чтобы включить это в сеансе проверки, вызовите метод `collectTaxIds` при создании сеанса:

    $checkout = $user->collectTaxIds()->checkout('price_tshirt');

При вызове этого метода клиенту будет доступен новый флажок, который позволяет ему указать, совершает ли он покупку от имени компании. Если это так, у них будет возможность указать свой идентификационный номер налогоплательщика.

> [!WARNING]
> Если вы уже настроили [автоматический сбор налогов](#tax-configuration) у сервис-провайдера вашего приложения, то эта функция будет включена автоматически, и нет необходимости вызывать метод `collectTaxIds`.

<a name="guest-checkouts"></a>
### Оформление заказа для гостей

С помощью метода `Checkout::guest` вы можете инициировать сеансы оформления заказа для гостей вашего приложения, которые не имеют "аккаунта":

    use Illuminate\Http\Request;
    use Laravel\Cashier\Checkout;

    Route::get('/product-checkout', function (Request $request) {
        return Checkout::guest()->create('price_tshirt', [
            'success_url' => route('your-success-route'),
            'cancel_url' => route('your-cancel-route'),
        ]);
    });

Подобно созданию сеансов оформления заказа для существующих пользователей, вы можете использовать дополнительные методы, доступные в экземпляре `Laravel\Cashier\CheckoutBuilder`, чтобы настроить сеанс оформления заказа для гостя:

    use Illuminate\Http\Request;
    use Laravel\Cashier\Checkout;

    Route::get('/product-checkout', function (Request $request) {
        return Checkout::guest()
            ->withPromotionCode('promo-code')
            ->create('price_tshirt', [
                'success_url' => route('your-success-route'),
                'cancel_url' => route('your-cancel-route'),
            ]);
    });

После завершения оформления заказа для гостя Stripe может отправить событие веб-хука `checkout.session.completed`, поэтому убедитесь, что [настроили веб-хук Stripe](https://dashboard.stripe.com/webhooks), чтобы фактически отправить это событие в ваше приложение. Как только веб-хук будет включен в панели управления Stripe, вы сможете [обработать веб-хук с помощью Cashier](#handling-stripe-webhooks). Объект, содержащийся в полезной нагрузке веб-хука, будет [`объектом оформления заказа`](https://stripe.com/docs/api/checkout/sessions/object), который вы можете проверить, чтобы выполнить заказ вашего клиента.

<a name="handling-failed-payments"></a>
## Обработка неудачных платежей

Иногда платежи за подписку или разовые платежи могут не выполняться. Когда это произойдет, Cashier выдаст исключение `Laravel\Cashier\Exceptions\IncompletePayment`, которое информирует вас о том, что это произошло. После перехвата этого исключения у вас есть два варианта дальнейших действий.

Во-первых, вы могли бы перенаправить своего клиента на специальную страницу подтверждения платежа, которая входит в комплект поставки Cashier. На этой странице уже есть связанный именованный маршрут, зарегистрированный через сервис-провайдера Cashier. Таким образом, вы можете перехватить исключение `IncompletePayment` и перенаправить пользователя на страницу подтверждения платежа:

    use Laravel\Cashier\Exceptions\IncompletePayment;

    try {
        $subscription = $user->newSubscription('default', 'price_monthly')
                                ->create($paymentMethod);
    } catch (IncompletePayment $exception) {
        return redirect()->route(
            'cashier.payment',
            [$exception->payment->id, 'redirect' => route('home')]
        );
    }

На странице подтверждения оплаты клиенту будет предложено повторно ввести данные своей кредитной карты и выполнить любые дополнительные действия, требуемые Stripe, такие как подтверждение "3D Secure". После подтверждения оплаты пользователь будет перенаправлен на URL, указанный указанным выше параметром `redirect`. При перенаправлении к URL-адресу будут добавлены строковые переменные запроса `message` (строка) и `success` (целое число). Страница оплаты в настоящее время поддерживает следующие типы способов оплаты:

<!-- <div class="content-list" markdown="1"> -->

- Credit Cards
- Alipay
- Bancontact
- BECS Direct Debit
- EPS
- Giropay
- iDEAL
- SEPA Direct Debit

<!-- </div> -->

В качестве альтернативы вы могли бы позволить Stripe обработать подтверждение платежа за вас. В этом случае вместо перенаправления на страницу подтверждения оплаты вы можете [настроить автоматические электронные письма Stripe для выставления счетов](https://dashboard.stripe.com/account/billing/automatic) на вашей панели управления Stripe. Однако, если будет обнаружено исключение `IncompletePayment`, вам все равно следует сообщить пользователю, что он получит электронное письмо с дальнейшими инструкциями по подтверждению платежа.

Исключения для оплаты могут быть созданы для следующих методов: `charge`, `invoiceFor` и `invoice` в моделях, использующих трейт `Billable`. При взаимодействии с подписками метод `create` в `SubscriptionBuilder`, а также методы `incrementAndInvoice` и `swapAndInvoice` в моделях `Subscription` и `SubscriptionItem` могут вызывать исключения неполной оплаты.

Определение того, имеет ли существующая подписка неполную оплату, может быть выполнено с помощью метода `hasIncompletePayment` в оплачиваемой модели или экземпляре подписки:

    if ($user->hasIncompletePayment('default')) {
        // ...
    }

    if ($user->subscription('default')->hasIncompletePayment()) {
        // ...
    }

Вы можете получить конкретный статус неполного платежа, проверив свойство `payment` в экземпляре исключения:

    use Laravel\Cashier\Exceptions\IncompletePayment;

    try {
        $user->charge(1000, 'pm_card_threeDSecure2Required');
    } catch (IncompletePayment $exception) {
        // Get the payment intent status...
        $exception->payment->status;

        // Check specific conditions...
        if ($exception->payment->requiresPaymentMethod()) {
            // ...
        } elseif ($exception->payment->requiresConfirmation()) {
            // ...
        }
    }

<a name="confirming-payments"></a>
### Подтверждение платежей

Некоторые методы оплаты требуют дополнительных данных для подтверждения платежей. Например, методы оплаты SEPA требуют дополнительных данных о "доверенности" во время процесса оплаты. Вы можете предоставить эти данные Cashier, используя метод `withPaymentConfirmationOptions`:

    $subscription->withPaymentConfirmationOptions([
        'mandate_data' => '...',
    ])->swap('price_xxx');

Вы можете ознакомиться с [документацией Stripe API](https://stripe.com/docs/api/payment_intents/confirm), чтобы просмотреть все параметры, принимаемые при подтверждении платежей.

<a name="strong-customer-authentication"></a>
## Аутентификация клиентов(SCA)

Если ваш бизнес или один из ваших клиентов базируется в Европе, вам необходимо будет соблюдать правила строгой аутентификации клиентов ЕС (SCA). Эти правила были введены Европейским союзом в сентябре 2019 года для предотвращения мошенничества с платежами. К счастью, Stripe и Cashier подготовлены для создания приложений, совместимых с SCA.

> [!WARNING]
> Прежде чем приступить к работе, ознакомьтесь с [руководством Stripe по PSD2 и SCA](https://stripe.com/guides/strong-customer-authentication), а также их [документация по новым SCA API](https://stripe.com/docs/strong-customer-authentication).

<a name="payments-requiring-additional-confirmation"></a>
### Платежи, требующие дополнительного подтверждения

Правила SCA часто требуют дополнительной верификации для подтверждения и обработки платежа. Когда это произойдет, Cashier выдаст исключение `Laravel\Cashier\Exceptions\IncompletePayment`, которое информирует вас о необходимости дополнительной проверки. Более подробную информацию о том, как обрабатывать эти исключения, можно найти в документации по [обработке неудачных платежей](#handling-failed-payments).

Экраны подтверждения оплаты, предоставляемые Stripe или Cashier, могут быть адаптированы к платежному потоку конкретного банка или эмитента карты и могут включать дополнительное подтверждение карты, временную небольшую плату, отдельную аутентификацию устройства или другие формы проверки.

<a name="incomplete-and-past-due-state"></a>
#### Неполное и просроченное состояние

Когда платеж нуждается в дополнительном подтверждении, подписка останется в состоянии `incomplete` или `past_due`, как указано в столбце базы данных `stripe_status`. Cashier автоматически активирует подписку клиента, как только будет завершено подтверждение оплаты и Stripe уведомит вашу заявку через веб-хук о ее завершении.

Для получения дополнительной информации о состояниях `incomplete` и `past_due`, пожалуйста, обратитесь к [нашей дополнительной документации по этим состояниям](#incomplete-and-past-due-status).

<a name="off-session-payment-notifications"></a>
### Уведомления об оплате вне сессии

Поскольку правила SCA требуют, чтобы клиенты время от времени проверяли свои платежные реквизиты, даже когда их подписка активна, Cashier может отправить клиенту уведомление, когда требуется подтверждение платежа вне сеанса. Например, это может произойти при обновлении подписки. Уведомление Cashier об оплате можно включить, установив переменную среды `CASHIER_PAYMENT_NOTIFICATION` в класс уведомлений. По умолчанию это уведомление отключено. Конечно, Cashier включает класс уведомлений, который вы можете использовать для этой цели, но при желании вы можете предоставить свой собственный класс уведомлений:

```ini
CASHIER_PAYMENT_NOTIFICATION=Laravel\Cashier\Notifications\ConfirmPayment
```

Чтобы убедиться, что уведомления о подтверждении оплаты вне сеанса доставляются, убедитесь, что [веб-хуки Stripe настроены](#handling-stripe-webhooks) для вашего приложения и веб-хук `invoice.payment_action_required` включен на вашей панели мониторинга Stripe. Кроме того, ваша `оплачиваемая` модель также должна использовать трейт Laravel `Illuminate\Notifications\Notifiable`.

> [!WARNING]
> Уведомления будут отправляться даже тогда, когда клиенты вручную производят платеж, требующий дополнительного подтверждения. К сожалению, у Stripe нет возможности узнать, что платеж был произведен вручную или "вне сеанса". Но клиент просто увидит сообщение "Платеж выполнен успешно", если он зайдет на страницу оплаты после того, как уже подтвердил свой платеж. Клиенту не будет разрешено случайно подтвердить один и тот же платеж дважды и понести случайную повторную оплату.

<a name="stripe-sdk"></a>
## Stripe SDK

Многие объекты Cashier's являются оболочками вокруг объектов Stripe SDK. Если вы хотите взаимодействовать с объектами Stripe напрямую, вы можете удобно извлечь их, используя метод `asStripe`:

    $stripeSubscription = $subscription->asStripeSubscription();

    $stripeSubscription->application_fee_percent = 5;

    $stripeSubscription->save();

Вы также можете использовать метод `updateStripeSubscription` для непосредственного обновления подписки Stripe:

    $subscription->updateStripeSubscription(['application_fee_percent' => 5]);

Вы можете вызвать метод `stripe` в классе `Cashier`, если хотите напрямую использовать клиент `Stripe\StripeClient`. Например, вы могли бы использовать этот метод для доступа к экземпляру `StripeClient` и получения списка тарифов из вашей учетной записи Stripe:

    use Laravel\Cashier\Cashier;

    $prices = Cashier::stripe()->prices->all();

<a name="testing"></a>
## Тестирование

При тестировании приложения, использующего Cashier, вы можете имитировать фактические HTTP-запросы к Stripe API; однако для этого вам потребуется частично повторно реализовать собственное поведение Cashier. Поэтому мы рекомендуем разрешить вашим тестам использовать фактический Stripe API. Хотя это происходит медленнее, это обеспечивает большую уверенность в том, что ваше приложение работает должным образом, и любые медленные тесты могут быть размещены в их собственной группе тестирования Pest / PHPUnit.

При тестировании помните, что у самого Cashier уже есть отличный набор тестов, поэтому вам следует сосредоточиться только на тестировании подписки и потока платежей вашего собственного приложения, а не на каждом базовом поведении Cashier.

Чтобы начать, добавьте **тестовую** версию вашего Stripe секрета в свой файл `phpunit.xml`:

```html
<env name="STRIPE_SECRET" value="sk_test_<your-key>"/>
```

Теперь, всякий раз, когда вы взаимодействуете с Cashier во время тестирования, он будет отправлять фактические запросы API в вашу среду тестирования Stripe. Для удобства вам следует предварительно заполнить свой тестовый аккаунт Stripe подписками / тарифами, которые вы можете использовать во время тестирования.

> [!NOTE]
> Для тестирования различных сценариев выставления счетов, таких как отказы в оплате по кредитной карте, вы можете использовать широкий спектр [тестовых номеров карт и токенов](https://stripe.com/docs/testing), предоставленный Stripe.
