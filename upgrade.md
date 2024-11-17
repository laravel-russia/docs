---
git: 1ae07b0e1cb7539b7c9c0f4a69eabbb93d50a3a0
---

# Руководство по обновлению

- [Обновление с 10.0 версии до 11.x](#upgrade-11.0)

<a name="high-impact-changes"></a>
## Изменения, оказывающие большое влияние

<!-- <div class="content-list" markdown="1"> -->

- [Обновление зависимостей](#updating-dependencies)
- [Структура приложения](#application-structure)
- [Типы с плавающей точкой](#floating-point-types)
- [Изменение столбцов](#modifying-columns)
- [Минимальная версия SQLite](#sqlite-minimum-version)
- [Обновление Sanctum](#updating-sanctum)

<!-- </div> -->

<a name="medium-impact-changes"></a>
## Изменения со средней степенью воздействия

<!-- <div class="content-list" markdown="1"> -->

- [Carbon 3](#carbon-3)
- [Рехешинг пароля](#password-rehashing)
- [Посекундное ограничение](#per-second-rate-limiting)
- [Пакет Spatie Once](#spatie-once-package)

<!-- </div> -->

<a name="low-impact-changes"></a>
###### Изменения с низким уровнем воздействия

<!-- <div class="content-list" markdown="1"> -->

- [Удаление Doctrine DBAL](#doctrine-dbal-removal)
- [Метод `casts` Eloquent-модели](#eloquent-model-casts-method)
- [Пространственные типы](#spatial-types)
- [Контракт `Enumerable`](#the-enumerable-contract)
- [Контракт `UserProvider`](#the-user-provider-contract)
- [Контракт `Authenticatable`](#the-authenticatable-contract)

<!-- </div> -->

<a name="upgrade-11.0"></a>
## Обновление с 10.0 версии до 11.x

<a name="estimated-upgrade-time-??-minutes"></a>
#### Приблизительное время обновления: 15 минут

> [!NOTE]
> Мы стараемся задокументировать каждое возможное изменение, которое может привести к нарушению совместимости. Поскольку некоторые из этих критических изменений находятся в малоизвестных частях фреймворка, только часть этих изменений может повлиять на ваше приложение. Хотите сэкономить время? Вы можете использовать [Laravel Shift](https://laravelshift.com/) , чтобы автоматизировать процесс обновления вашего приложения.

<a name="updating-dependencies"></a>
### Обновление зависимостей

**Вероятность воздействия: высокая**

#### Требуется PHP 8.2.0

Laravel теперь требует PHP версии 8.2.0 или выше.

#### Требуется curl 7.34.0

HTTP-клиенту Laravel теперь требуется версия Curl 7.34.0 или выше.

#### Зависимости Composer

Обновите следующие зависимости в вашем файле `composer.json`:

<!-- <div class="content-list" markdown="1"> -->

- `laravel/framework` to `^11.0`
- `nunomaduro/collision` to `^8.1`
- `laravel/breeze` to `^2.0` (если установлено)
- `laravel/cashier` to `^15.0` (если установлено)
- `laravel/dusk` to `^8.0` (если установлено)
- `laravel/jetstream` to `^5.0` (если установлено)
- `laravel/octane` to `^2.3` (если установлено)
- `laravel/passport` to `^12.0` (если установлено)
- `laravel/sanctum` to `^4.0` (если установлено)
- `laravel/scout` to `^10.0` (если установлено)
- `laravel/spark-stripe` to `^5.0` (если установлено)
- `laravel/telescope` to `^5.0` (если установлено)
- `livewire/livewire` to `^3.4` (если установлено)
- `inertiajs/inertia-laravel` to `^1.0` (если установлено)

<!-- </div> -->

Если ваше приложение использует Laravel Cashier Stripe, Passport, Sanctum, Spark Stripe или Telescope, вам необходимо опубликовать их миграции в ваше приложение. Cashier Stripe, Passport, Sanctum, Spark Stripe и Telescope **больше не загружают автоматически миграции из собственного каталога миграций**. Поэтому вам следует запустить следующую команду, чтобы опубликовать их миграции в вашем приложении:

```bash
php artisan vendor:publish --tag=cashier-migrations
php artisan vendor:publish --tag=passport-migrations
php artisan vendor:publish --tag=sanctum-migrations
php artisan vendor:publish --tag=spark-migrations
php artisan vendor:publish --tag=telescope-migrations
```

Кроме того, вам следует просмотреть руководства по обновлению для каждого из этих пакетов, чтобы быть в курсе любых дополнительных критических изменений:

- [Laravel Cashier Stripe](#cashier-stripe)
- [Laravel Passport](#passport)
- [Laravel Sanctum](#sanctum)
- [Laravel Spark Stripe](#spark-stripe)
- [Laravel Telescope](#telescope)

Если вы установили установщик Laravel вручную, вам следует обновить установщик через Composer:

```bash
composer global require laravel/installer:^5.6
```

Наконец, вы можете удалить зависимость Composer `doctrine/dbal`, если вы ранее добавили ее в свое приложение, поскольку Laravel больше не зависит от этого пакета.

<a name="application-structure"></a>
### Структура приложения

Laravel 11 представляет новую структуру приложения с меньшим количеством файлов по умолчанию. А именно, новые приложения Laravel содержат меньше поставщиков услуг, посредников и файлов конфигурации.

Однако мы **не рекомендуем** приложениям Laravel 10, обновляющимся до Laravel 11, пытаться перенести структуру своих приложений, поскольку Laravel 11 был тщательно настроен для поддержки структуры приложений Laravel 10.

<a name="authentication"></a>
### Аутентификация

<a name="password-rehashing"></a>
#### Рехешинг пароля

**Вероятность воздействия: Низкая**

Laravel 11 автоматически перехэширует пароли вашего пользователя во время аутентификации, если «рабочий фактор» вашего алгоритма хеширования был обновлен с момента последнего хеширования пароля.

Обычно это не должно нарушать работу вашего приложения; однако, если поле «password» вашей модели `User` имеет имя, отличное от `password`, вам следует указать имя поля через свойство `authPasswordName` модели:

    protected $authPasswordName = 'custom_password_field';

Так же вы можете отключить перехэширование пароля, добавив параметр `rehash_on_login` в файл конфигурации вашего приложения `config/hashing.php`:

    'rehash_on_login' => false,

<a name="the-user-provider-contract"></a>
#### Контракт `UserProvider`

**Вероятность воздействия: Низкая**

Контракт `Illuminate\Contracts\Auth\UserProvider` получил новый метод `rehashPasswordIfRequired`. Этот метод отвечает за повторное хэширование и сохранение пароля пользователя в хранилище при изменении коэффициента работы алгоритма хеширования приложения.

Если ваше приложение или пакет определяет класс, реализующий этот интерфейс, вам следует добавить в вашу реализацию новый метод `rehashPasswordIfRequired`. Эталонную реализацию можно найти в классе `Illuminate\Auth\EloquentUserProvider`:

```php
public function rehashPasswordIfRequired(Authenticatable $user, array $credentials, bool $force = false);
```

<a name="the-authenticatable-contract"></a>
#### Контракт `Authenticatable`

**Вероятность воздействия: Низкая**

Контракт `Illuminate\Contracts\Auth\Authenticatable` получил новый метод `getAuthPasswordName`. Этот метод отвечает за возврат имени столбца пароля вашего аутентифицируемого объекта.

Если ваше приложение или пакет определяет класс, реализующий этот интерфейс, вам следует добавить в вашу реализацию новый метод `getAuthPasswordName`:

```php
public function getAuthPasswordName()
{
    return 'password';
}
```

Модель `User` по умолчанию, включенная в Laravel, автоматически получает этот метод, поскольку этот метод включен в трейт `Illuminate\Auth\Authenticatable`.

<a name="the-authentication-exception-class"></a>
#### Класс `AuthenticationException`

**Вероятность воздействия: Очень низкая**

Метод `redirectTo` класса `Illuminate\Auth\AuthenticationException` теперь требует экземпляр `Illuminate\Http\Request` в качестве первого аргумента. Если вы вручную перехватываете это исключение и вызываете метод `redirectTo`, вам следует соответствующим образом обновить свой код:

```php
if ($e instanceof AuthenticationException) {
    $path = $e->redirectTo($request);
}
```

<a name="email-verification-notification-on-registration"></a>
#### Уведомление о проверке электронной почты при регистрации

**Вероятность воздействия: Очень низкая**

Слушатель `SendEmailVerificationNotification` теперь автоматически регистрируется для события `Registered`, если он еще не зарегистрирован в `EventServiceProvider` вашего приложения. Если `EventServiceProvider` вашего приложения не регистрирует этого слушателя и вы не хотите, чтобы Laravel автоматически регистрировал его для вас, вам следует определить пустой метод `configureEmailVerification` в `EventServiceProvider` вашего приложения:

```php
protected function configureEmailVerification()
{
    // ...
}
```

<a name="cache"></a>
### Кеш

<a name="cache-key-prefixes"></a>
#### Префиксы ключей кэша

**Вероятность воздействия: Очень низкая**

Раньше, если префикс ключа кэша был определен для хранилищ кэша DynamoDB, Memcached или Redis, Laravel добавлял к префиксу `:`. В Laravel 11 префикс ключа кэша не получает суффикс `:`. Если вы хотите сохранить предыдущее поведение префиксов, вы можете вручную добавить суффикс `:` к префиксу ключа кэша.

<a name="collections"></a>
### Коллекции

<a name="the-enumerable-contract"></a>
#### Контракт `Enumerable`

**Вероятность воздействия: Низкая**

Метод `dump` контракта `Illuminate\Support\Enumerable` был обновлен, чтобы принимать переменный аргумент `...$args`. Если вы реализуете этот интерфейс, вам следует соответствующим образом обновить свою реализацию:

```php
public function dump(...$args);
```

<a name="database"></a>
### База данных

<a name="sqlite-minimum-version"></a>
#### SQLite 3.26.0+

**Вероятность воздействия: Высокая**

Если ваше приложение использует базу данных SQLite, требуется SQLite 3.26.0 или более поздняя версия.

<a name="eloquent-model-casts-method"></a>
#### Метод `casts` Eloquent-модели

**Вероятность воздействия: низкая**

Базовый класс модели Eloquent теперь определяет метод `casts` для поддержки определения приведения атрибутов. Если одна из моделей вашего приложения определяет отношение приведения, это может конфликтовать с методом `casts`, который теперь присутствует в базовом классе модели Eloquent.

<a name="modifying-columns"></a>
#### Изменение столбцов

**Вероятность воздействия: Высокая**

При изменении столбца теперь вы должны явно включать все модификаторы, которые вы хотите сохранить в определении столбца после его изменения. Любые недостающие атрибуты будут удалены. Например, чтобы сохранить атрибуты `unsigned`, `default` и `comment`, вы должны явно вызывать каждый модификатор при изменении столбца, даже если эти атрибуты были назначены столбцу при предыдущей миграции.

Например, представьте, что у вас есть миграция, в результате которой создается столбец `votes` с атрибутами `unsigned`, `default` и `comment`:

```php
Schema::create('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('The vote count');
});
```

Позже вы напишете миграцию, которая также изменит столбец на значение `nullable`:

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->nullable()->change();
});
```

В Laravel 10 эта миграция сохранит атрибуты `unsigned`, `default` и `comment` в столбце. Однако в Laravel 11 миграция теперь также должна включать все атрибуты, которые ранее были определены в столбце. В противном случае они будут удалены:

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')
        ->unsigned()
        ->default(1)
        ->comment('The vote count')
        ->nullable()
        ->change();
});
```

Метод `change` не меняет индексы столбца. Поэтому вы можете использовать модификаторы индекса, чтобы явно добавлять или удалять индекс при изменении столбца:

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```

Если вы не хотите обновлять все существующие миграции «изменений» в вашем приложении, чтобы сохранить существующие атрибуты столбца, вы можете просто [сократить свои миграции](/docs/{{version}}/migrations#squashing-migrations):

```bash
php artisan schema:dump
```

Как только ваши миграции будут завершены, Laravel «перенесет» базу данных, используя файл схемы вашего приложения, прежде чем запускать любые ожидающие миграции.

<a name="floating-point-types"></a>
#### Типы с плавающей точкой

**Вероятность воздействия: Высокая**

Типы столбцов миграции `double` и `float` были переписаны, чтобы обеспечить единообразие во всех базах данных.

Тип столбца `double` теперь создает эквивалентный столбец `DOUBLE` без общего количества цифр и мест (цифр после десятичной точки), что является стандартным синтаксисом SQL. Поэтому вы можете удалить аргументы для `$total` и `$places`:

```php
$table->double('amount');
```

Столбец типа `float` теперь создает эквивалентный столбец `FLOAT` без общего количества цифр и мест (цифр после десятичной точки), но с дополнительной спецификацией `$precision` для определения размера хранилища в виде 4-байтового столбца одинарной точности или 8-байтовый столбец двойной точности. Таким образом, вы можете удалить аргументы для `$total` и `$places` и указать необязательное значение `$precision` в соответствии с вашим желаемым значением и в соответствии с документацией вашей базы данных:

```php
$table->float('amount', precision: 53);
```

Методы `unsignedDecimal`, `unsignedDouble` и `unsignedFloat` были удалены, поскольку модификатор `unsigned` для этих типов столбцов устарел в MySQL и никогда не стандартизировался в других системах баз данных. Однако, если вы хотите и дальше использовать устаревший беззнаковый атрибут для этих типов столбцов, вы можете связать метод `unsigned` с определением столбца:

```php
$table->decimal('amount', total: 8, places: 2)->unsigned();
$table->double('amount')->unsigned();
$table->float('amount', precision: 53)->unsigned();
```

<a name="dedicated-mariadb-driver"></a>
#### Выделенный драйвер MariaDB

**Вероятность воздействия: Очень низкая**

Вместо того, чтобы всегда использовать драйвер MySQL при подключении к базам данных MariaDB, Laravel 11 добавляет специальный драйвер базы данных для MariaDB.

Если ваше приложение подключается к базе данных MariaDB, вы можете обновить конфигурацию подключения новым драйвером MariaDB, чтобы в будущем воспользоваться специальными функциями MariaDB:

    'driver' => 'mariadb',
    'url' => env('DB_URL'),
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    // ...

В настоящее время новый драйвер MariaDB ведет себя как текущий драйвер MySQL, за одним исключением: метод построения схемы `uuid` создает собственные столбцы `UUID` вместо столбцов `char(36)`.

Если в ваших существующих миграциях используется метод построения схемы `uuid`, и вы решили использовать новый драйвер базы данных `mariadb`, вам следует обновить вызовы метода `uuid` в вашей миграции на `char`, чтобы избежать критических изменений или неожиданного поведения:

```php
Schema::table('users', function (Blueprint $table) {
    $table->char('uuid', 36);.

    // ...
});
```

<a name="spatial-types"></a>
#### Пространственные типы

**Вероятность воздействия: Низкая**

Типы пространственных столбцов миграции базы данных были переписаны, чтобы обеспечить единообразие во всех базах данных. Поэтому вы можете удалить методы `point`, `lineString`, `polygon`, `geometryCollection`, `multiPoint`, `multiLineString`, `multiPolygon` и `multiPolygonZ` из своих миграций и использовать вместо этого методы `geometry` или `geography`:

```php
$table->geometry('shapes');
$table->geography('coordinates');
```

Чтобы явно ограничить тип или идентификатор системы пространственной привязки для значений, хранящихся в столбце в MySQL, MariaDB и PostgreSQL, вы можете передать методу `subtype` и `srid`:

```php
$table->geometry('dimension', subtype: 'polygon', srid: 0);
$table->geography('latitude', subtype: 'point', srid: 4326);
```

Модификаторы столбцов `isGeometry` и `projection` грамматики PostgreSQL были соответственно удалены.

<a name="doctrine-dbal-removal"></a>
#### Удаление Doctrine DBAL

**Вероятность воздействия: Низкая**

Следующий список классов и методов, связанных с Doctrine DBAL, был удален. Laravel больше не зависит от этого пакета, и регистрация пользовательских типов Doctrines больше не требуется для правильного создания и изменения различных типов столбцов, для которых ранее требовались пользовательские типы:

- `Illuminate\Database\Schema\Builder::$alwaysUsesNativeSchemaOperationsIfPossible` class property
- `Illuminate\Database\Schema\Builder::useNativeSchemaOperationsIfPossible()` method
- `Illuminate\Database\Connection::usingNativeSchemaOperations()` method
- `Illuminate\Database\Connection::isDoctrineAvailable()` method
- `Illuminate\Database\Connection::getDoctrineConnection()` method
- `Illuminate\Database\Connection::getDoctrineSchemaManager()` method
- `Illuminate\Database\Connection::getDoctrineColumn()` method
- `Illuminate\Database\Connection::registerDoctrineType()` method
- `Illuminate\Database\DatabaseManager::registerDoctrineType()` method
- `Illuminate\Database\PDO` directory
- `Illuminate\Database\DBAL\TimestampType` class
- `Illuminate\Database\Schema\Grammars\ChangeColumn` class
- `Illuminate\Database\Schema\Grammars\RenameColumn` class
- `Illuminate\Database\Schema\Grammars\Grammar::getDoctrineTableDiff()` method

Кроме того, больше не требуется регистрация пользовательских типов Doctrine через `dbal.types` в файле конфигурации `database` вашего приложения.

Если вы ранее использовали Doctrine DBAL для проверки вашей базы данных и связанных с ней таблиц, вы можете использовать новые собственные методы схемы Laravel (`Schema::getTables()`, `Schema::getColumns()`, `Schema::getIndexes()` `, `Schema::getForeignKeys()` и т. д.).

<a name="deprecated-schema-methods"></a>
#### Устаревшие методы схемы

**Вероятность воздействия: Очень низкая**

Устаревшие методы `Schema::getAllTables()`, `Schema::getAllViews()` и `Schema::getAllTypes()`, основанные на Doctrine, были удалены в пользу новых встроенных в Laravel `Schema::getTables()`. , `Schema::getViews()` и `Schema::getTypes()`.

При использовании PostgreSQL и SQL Server ни один из новых методов схемы не будет принимать ссылку из трех частей (например, `database.schema.table`). Поэтому вместо этого вам следует использовать `connection()` для объявления базы данных:

```php
Schema::connection('database')->hasTable('schema.table');
```

<a name="get-column-types"></a>
#### Метод построителя схемы `getColumnType()`

**Вероятность воздействия: Очень низкая**

Метод `Schema::getColumnType()` теперь всегда возвращает фактический тип данного столбца, а не эквивалентный тип Doctrine DBAL.

<a name="database-connection-interface"></a>
#### Интерфейс подключения к базе данных

**Вероятность воздействия: Очень низкая**

Интерфейс `Illuminate\Database\ConnectionInterface` получил новый метод `scalar`. Если вы определяете собственную реализацию этого интерфейса, вам следует добавить в вашу реализацию метод `scalar`:

```php
public function scalar($query, $bindings = [], $useReadPdo = true);
```

<a name="dates"></a>
### Даты

<a name="carbon-3"></a>
#### Carbon 3

**Вероятность воздействия: Средняя**

Laravel 11 поддерживает как Carbon 2, так и Carbon 3. Carbon — это библиотека манипулирования датами, широко используемая Laravel и пакетами во всей экосистеме. Если вы обновитесь до Carbon 3, имейте в виду, что методы `diffIn*` теперь возвращают числа с плавающей запятой и могут возвращать отрицательные значения для указания направления времени, что является существенным отличием от Carbon 2. Просмотрите [журнал изменений](https:/ /github.com/briannesbitt/Carbon/releases/tag/3.0.0) и [документацию](https://carbon.nesbot.com/docs/#api-carbon-3) Carbon для получения подробной информации о том, как обрабатывать эти и другие изменения.

<a name="mail"></a>
### Почта

<a name="the-mailer-contract"></a>
#### Контракт `Mailer`

**Вероятность воздействия: Очень низкая**

Контракт `Illuminate\Contracts\Mail\Mailer` получил новый метод `sendNow`. Если ваше приложение или пакет вручную реализует этот контракт, вам следует добавить в свою реализацию новый метод `sendNow`:

```php
public function sendNow($mailable, array $data = [], $callback = null);
```

<a name="packages"></a>
### Пакеты

<a name="publishing-service-providers"></a>
#### Публикация поставщиков услуг в приложении

**Вероятность воздействия: Очень низкая**

Если вы написали пакет Laravel, который вручную публикует поставщика услуг в каталоге приложения `app/Providers` и вручную изменяет файл конфигурации приложения `config/app.php` для регистрации поставщика услуг, вам следует обновить свой пакет, чтобы использовать новый метод `ServiceProvider::addProviderToBootstrapFile`.

Метод `addProviderToBootstrapFile` автоматически добавит опубликованного вами поставщика услуг в файл `bootstrap/providers.php` приложения, поскольку массив `providers` не существует в файле конфигурации `config/app.php` в новом приложения Laravel 11.

```php
use Illuminate\Support\ServiceProvider;

ServiceProvider::addProviderToBootstrapFile(Provider::class);
```

<a name="queues"></a>
### Очереди

<a name="the-batch-repository-interface"></a>
#### Интерфейс `BatchRepository`

**Вероятность воздействия: Очень низкая**

Интерфейс `Illuminate\Bus\BatchRepository` получил новый метод `rollBack`. Если вы реализуете этот интерфейс в своем собственном пакете или приложении, вам следует добавить в свою реализацию этот метод:

```php
public function rollBack();
```

<a name="synchronous-jobs-in-database-transactions"></a>
#### Синхронные задания в транзакциях базы данных

**Вероятность воздействия: Очень низкая**
 
Раньше синхронные задания (задания, использующие драйвер очереди `sync`) выполнялись немедленно, независимо от того, было ли для параметра конфигурации `after_commit` соединения с очередью установлено значение `true` или для задания был вызван метод `afterCommit`.

В Laravel 11 синхронные задания очереди теперь будут учитывать конфигурацию соединения очереди или задания «после фиксации».

<a name="rate-limiting"></a>
### Ограничение скорости

<a name="per-second-rate-limiting"></a>
#### Посекундное ограничение

**Вероятность воздействия: Средняя**

Laravel 11 поддерживает посекундное ограничение скорости вместо поминутной детализации. Существует множество потенциальных критических изменений, о которых вам следует знать, связанных с этим изменением.

Конструктор класса `GlobalLimit` теперь принимает секунды вместо минут. Этот класс не документирован и обычно не будет использоваться вашим приложением:

```php
new GlobalLimit($attempts, 2 * 60);
```

Конструктор класса `Limit` теперь принимает секунды вместо минут. Все документированные варианты использования этого класса ограничиваются статическими конструкторами, такими как `Limit::perMinute` и `Limit::perSecond`. Однако если вы создаете экземпляр этого класса вручную, вам следует обновить приложение, чтобы предоставить секунды конструктору класса:

```php
new Limit($key, $attempts, 2 * 60);
```

Свойство `decayMinutes` класса `Limit` было переименовано в `decaySeconds` и теперь содержит секунды вместо минут.

Конструкторы классов `Illuminate\Queue\Middleware\ThrottlesExceptions` и `Illuminate\Queue\Middleware\ThrottlesExceptionsWithRedis` теперь принимают секунды вместо минут:

```php
new ThrottlesExceptions($attempts, 2 * 60);
new ThrottlesExceptionsWithRedis($attempts, 2 * 60);
```

<a name="cashier-stripe"></a>
### Cashier Stripe

<a name="updating-cashier-stripe"></a>
#### Обновление Cashier Stripe

**Вероятность воздействия: Высокая**

Laravel 11 больше не поддерживает Cashier Stripe 14.x. Поэтому вам следует обновить зависимость Laravel Cashier Stripe вашего приложения до `^15.0` в вашем файле `composer.json`.

Cashier Stripe 15.0 больше не загружает миграции автоматически из собственного каталога миграций. Вместо этого вам следует запустить следующую команду, чтобы опубликовать миграции Cashier Stripe в ваше приложение:

```shell
php artisan vendor:publish --tag=cashier-migrations
```

Ознакомьтесь с полным [руководством по обновлению Cashier Stripe](https://github.com/laravel/cashier-stripe/blob/15.x/UPGRADE.md), чтобы узнать о дополнительных критических изменениях.

<a name="spark-stripe"></a>
### Spark (Stripe)

<a name="updating-spark-stripe"></a>
#### Обновление Spark Stripe

**Вероятность воздействия: Высокая**

Laravel 11 больше не поддерживает Laravel Spark Stripe 4.x. Поэтому вам следует обновить зависимость Laravel Spark Stripe вашего приложения до `^5.0` в файле `composer.json`.

Spark Stripe 5.0 больше не загружает миграции автоматически из собственного каталога миграций. Вместо этого вам следует запустить следующую команду, чтобы опубликовать миграции Spark Stripe в ваше приложение:

```shell
php artisan vendor:publish --tag=spark-migrations
```

Ознакомьтесь с полным [руководством по обновлению Spark Stripe](https://spark.laravel.com/docs/spark-stripe/upgrade.html), чтобы узнать о дополнительных критических изменениях.

<a name="passport"></a>
### Passport

<a name="updating-telescope"></a>
#### Обновление Passport

**Вероятность воздействия: Высокая**

Laravel 11 больше не поддерживает Laravel Passport 11.x. Поэтому вам следует обновить зависимость Laravel Passport вашего приложения до `^12.0` в вашем файле `composer.json`.

Passport 12.0 больше не загружает миграции автоматически из собственного каталога миграций. Вместо этого вам следует запустить следующую команду, чтобы опубликовать миграции Passport в вашем приложении:

```shell
php artisan vendor:publish --tag=passport-migrations
```

Кроме того, тип предоставления пароля по умолчанию отключен. Вы можете включить его, вызвав метод `enablePasswordGrant` в методе `boot` `AppServiceProvider` вашего приложения:

    public function boot(): void
    {
        Passport::enablePasswordGrant();
    }

<a name="sanctum"></a>
### Sanctum

<a name="updating-sanctum"></a>
#### Обновление Sanctum

**Вероятность воздействия: Высокая**

Laravel 11 больше не поддерживает Laravel Sanctum 3.x. Поэтому вам следует обновить зависимость Laravel Sanctum вашего приложения до `^4.0` в вашем файле `composer.json`.

Sanctum 4.0 больше не загружает миграции автоматически из собственного каталога миграций. Вместо этого вам следует запустить следующую команду, чтобы опубликовать миграции Sanctum в вашем приложении:

```shell
php artisan vendor:publish --tag=sanctum-migrations
```

Затем в файле конфигурации вашего приложения `config/sanctum.php` вам следует обновить ссылки на посредников `authenticate_session`, `encrypt_cookies` и `validate_csrf_token` следующим образом:

    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],

<a name="telescope"></a>
### Telescope

<a name="updating-telescope"></a>
#### Обновление Telescope

**Вероятность воздействия: Высокая**

Laravel 11 больше не поддерживает Laravel Telescope 4.x. Поэтому вам следует обновить зависимость Laravel Telescope вашего приложения до `^5.0` в вашем файле `composer.json`.

Telescope 5.0 больше не загружает миграции автоматически из собственного каталога миграций. Вместо этого вам следует запустить следующую команду, чтобы опубликовать миграции Telescope в вашем приложении:

```shell
php artisan vendor:publish --tag=telescope-migrations
```

<a name="spatie-once-package"></a>
### Пакет Spatie Once

**Вероятность воздействия: Средняя**

Laravel 11 теперь предоставляет собственную функцию [`once`](/docs/{{version}}/helpers#method-once), чтобы гарантировать, что данное замыкание будет выполнено только один раз. Поэтому, если ваше приложение зависит от пакета spatie/once, вам следует удалить его из файла `composer.json` вашего приложения, чтобы избежать конфликтов.

<a name="miscellaneous"></a>
### Разное

Мы также рекомендуем вам просмотреть изменения в `laravel/laravel` [репозиторий GitHub](https://github.com/laravel/laravel). Хотя многие из этих изменений не обязательны, вы можете захотеть синхронизировать эти файлы с вашим приложением. Некоторые из этих изменений будут описаны в этом руководстве по обновлению, а другие, например изменения в файлах конфигурации или комментариях, не будут рассмотрены. Вы можете легко просмотреть изменения с помощью [инструмента сравнения GitHub](https://github.com/laravel/laravel/compare/10.x...11.x) и выбрать, какие обновления важны для вас.
