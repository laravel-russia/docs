---
git: 46c2634ef5a4f15427c94a3157b626cf5bd3937f
---

# База данных · Начало работы


<a name="introduction"></a>
## Введение

Почти каждое современное веб-приложение взаимодействует с базой данных. Laravel делает работу с базами данных 
чрезвычайно простой благодаря поддержке множества баз данных, используя либо 
"сырой" SQL [построителя запросов](/docs/{{version}}/queries), либо [Eloquent ORM](/docs/{{version}}/eloquent). 
В настоящее время Laravel обеспечивает поддержку пяти баз данных:

<!-- <div class="content-list" markdown="1"> -->
- MariaDB 10.10+ ([Version Policy](https://mariadb.org/about/#maintenance-policy))
- MySQL 5.7+ ([Version Policy](https://en.wikipedia.org/wiki/MySQL#Release_history))
- PostgreSQL 11.0+ ([Version Policy](https://www.postgresql.org/support/versioning/))
- SQLite 3.8.8+
- SQL Server 2017+ ([Version Policy](https://docs.microsoft.com/en-us/lifecycle/products/?products=sql-server))
<!-- </div> -->

<a name="configuration"></a>
### Настройки

Конфигурация служб баз данных Laravel находится в конфигурационном файле `config/database.php` вашего приложения. 
В этом файле вы можете определить все соединения к базе данных, а также указать, какое соединение должно использоваться 
по умолчанию. Большинство параметров конфигурации в этом файле определяется значениями переменных окружения вашего приложения. 
В этом файле представлены примеры для большинства систем баз данных, поддерживаемых Laravel.

Параметры для настройки баз данных находятся в файле `config/database.php`. В файле можно указать несколько подключений 
к базам данных, а также указать, какое из подключений должно использоваться по умолчаний. Большинство параметров в файле 
определяется значениями переменных из `.env` файла вашего приложения.

По умолчанию файл `.env.example` для [конфигурации окружения](/docs/{{version}}/configuration#environment-configuration) Laravel готов к использованию с [Laravel Sail](/docs/{{version}}/sail), 
который представляет собой сборку Docker для разработки приложений Laravel на вашем локальном компьютере. 
Однако вы можете изменить настройки базы данных по мере необходимости для своей локальной базы данных.

<a name="sqlite-configuration"></a>
#### Конфигурация SQLite

Базы данных SQLite содержатся в одном файле вашей файловой системы. Вы можете создать новую базу данных SQLite, 
используя команду `touch` в консоли: `touch database/database.sqlite`. После создания базы данных вы можете легко 
настроить переменные окружения так, чтобы они указывали на эту базу данных, указав абсолютный путь к базе данных в 
переменной `DB_DATABASE` окружения:

```ini
DB_CONNECTION=sqlite
DB_DATABASE=/absolute/path/to/database.sqlite
```

Чтобы включить ограничения внешнего ключа для соединений SQLite, установите переменную `DB_FOREIGN_KEYS` окружения в `true`:

```ini
DB_FOREIGN_KEYS=true
```

<a name="mssql-configuration"></a>
#### Настройка Microsoft SQL Server

Чтобы использовать базу данных Microsoft SQL Server, вы должны убедиться, что у вас установлены расширения PHP 
`sqlsrv` и `pdo_sqlsrv`, а также любые зависимости, которые могут им потребоваться, например, драйвер Microsoft SQL ODBC.

<a name="configuration-using-urls"></a>
#### Настройка с использованием URL

Обычно соединения с базой данных конфигурируются с использованием нескольких значений, 
таких как `host`, `database`, `username`, `password` и т.д. Каждое из этих значений имеет свою собственную соответствующую 
переменную окружения. Это означает, что при указании информации о соединении с базой данных на рабочем web-сервере вам 
необходимо управлять несколькими переменными окружения.

Некоторые поставщики СУБД, такие, как AWS и Heroku, предоставляют единый «URL» базы данных, который содержит всю 
информацию о соединении в одной строке. Пример URL-адреса базы данных может выглядеть так:

```html
mysql://root:password@127.0.0.1/forge?charset=UTF-8
```

Эти URL обычно следуют соглашению стандартной схемы:

```html
driver://username:password@host:port/database?options
```

Для удобства Laravel поддерживает эти URL-адреса в качестве альтернативы настройке базы данных с несколькими параметрами 
конфигурации. Если присутствует параметр конфигурации `url` (или соответствующая переменная `DATABASE_URL` окружения), 
то он будет использоваться для получения информации о соединении с базой данных и об учетных данных.

<a name="read-and-write-connections"></a>
### Соединения для чтения и записи

По желанию можно использовать одно соединение с базой данных для операторов `SELECT`, а другое – для операторов 
`INSERT`, `UPDATE` и `DELETE`. Laravel упрощает эту задачу, и всегда будут использоваться соответствующие соединения, 
независимо от того, используете ли вы сырые запросы построителя запросов или Eloquent ORM.

Чтобы увидеть, как должны быть настроены соединения для чтения/записи, давайте посмотрим на этот пример:

    'mysql' => [
        'read' => [
            'host' => [
                '192.168.1.1',
                '196.168.1.2',
            ],
        ],
        'write' => [
            'host' => [
                '196.168.1.3',
            ],
        ],
        'sticky' => true,
        'driver' => 'mysql',
        'database' => 'database',
        'username' => 'root',
        'password' => '',
        'charset' => 'utf8mb4',
        'collation' => 'utf8mb4_unicode_ci',
        'prefix' => '',
    ],

Обратите внимание, что в массив конфигурации были добавлены три ключа: `read`, `write` и `sticky`. 
Ключи `read` и `write` имеют значения массива, содержащие один ключ: `host`. Остальные параметры базы данных 
для соединений `read` и `write` будут объединены из основного массива конфигурации `mysql`.

В массивы `read` и `write` вам нужно помещать только те элементы, значения которых вы хотите переопределить из основного 
массива `mysql`. Таким образом, в этом случае `192.168.1.1` будет использоваться в качестве хоста для соединения «чтение», 
а `192.168.1.3` – для соединения «запись». Учетные данные БД, префикс, набор символов и все другие параметры из основного 
массива `mysql` будут совместно использоваться обоими соединениями. Если в массиве конфигурации `host` существует 
несколько значений, то для каждого запроса хост базы данных будет выбран случайным образом.

<a name="the-sticky-option"></a>
#### Параметр `sticky`

Параметр `sticky` – это *необязательное* значение, которое может использоваться для разрешения немедленного чтения 
записей, которые были записаны в базу данных во время текущего цикла запроса. Если опция `sticky` включена и в текущем 
цикле запроса к базе данных была выполнена операция «записи», то любые дальнейшие операции «чтения» будут использовать 
соединение «запись». Это гарантирует, что любые данные, записанные во время цикла запроса, могут быть немедленно обратно 
прочитаны из базы данных во время того же запроса. Вам решать, является ли это желаемым поведением для вашего приложения.

<a name="running-queries"></a>
## Выполнение SQL-запросов

После того как вы настроили соединение с базой данных, вы можете выполнять запросы, используя фасад `DB`. 
Фасад `DB` содержит методы для каждого типа запроса: `select`, `update`, `insert`, `delete`, и `statement`.

<a name="running-a-select-query"></a>
#### Выполнение Select-запроса

Чтобы выполнить базовый запрос `SELECT`, вы можете использовать метод `select` фасада `DB`:

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Support\Facades\DB;
    use Illuminate\View\View;

    class UserController extends Controller
    {
        /**
         * Показать список всех пользователей приложения.
         */
        public function index(): View
        {
            $users = DB::select('select * from users where active = ?', [1]);

            return view('user.index', ['users' => $users]);
        }
    }

Первым аргументом, переданным методу `select`, является SQL-запрос, а вторым аргументом – параметры необходимые для запроса. 
Обычно это значения условий для `where`. Использование параметров необходимо для защиты от SQL-инъекций.

Метод `select` всегда возвращает «массив» результатов. Каждый результат в массиве будет объектом `stdClass` PHP, 
представляющим запись из базы данных:

    use Illuminate\Support\Facades\DB;

    $users = DB::select('select * from users');

    foreach ($users as $user) {
        echo $user->name;
    }

<a name="selecting-scalar-values"></a>
#### Выбор скалярных значений

Иногда ваш запрос к базе данных может вернуть единственное скалярное значение. Вместо того чтобы получать скалярный 
результат запроса из объекта записи, Laravel позволяет вам получать это значение напрямую с использованием метода `scalar`:
    
    $burgers = DB::scalar(
    "select count(case when food = 'burger' then 1 end) as burgers from menu"
    );

<a name="selecting-multiple-result-sets"></a>
#### Выбор нескольких наборов результатов

Если ваше приложение вызывает хранимые процедуры, возвращающие несколько наборов результатов, вы можете использовать 
метод `selectResultSets` для получения всех наборов результатов, возвращенных хранимой процедурой:

    [$options, $notifications] = DB::selectResultSets(
    "CALL get_user_options_and_notifications(?)", $request->user()->id
    );

<a name="using-named-bindings"></a>
#### Использование именованных псевдопеременных

Вместо использования символа `?` для связывания параметров вы можете выполнить запрос, используя именованные привязки:

    $results = DB::select('select * from users where id = :id', ['id' => 1]);

<a name="running-an-insert-statement"></a>
#### Выполнение Insert-запроса

Чтобы выполнить запрос с `INSERT`, вы можете использовать метод `insert` фасада `DB`. Как и `select`, этот метод 
принимает запрос SQL в качестве первого аргумента, а параметры – в качестве второго аргумента:

    use Illuminate\Support\Facades\DB;

    DB::insert('insert into users (id, name) values (?, ?)', [1, 'Marc']);

<a name="running-an-update-statement"></a>
#### Выполнение Update-запроса

Метод `update` следует использовать для обновления существующих записей в базе данных. Метод вернет количество подходящих
под условие строк:

    use Illuminate\Support\Facades\DB;

    $affected = DB::update(
        'update users set votes = 100 where name = ?',
        ['Anita']
    );

<a name="running-a-delete-statement"></a>
#### Выполнение Delete-запроса

Для удаления записей из базы данных следует использовать метод `delete`. Как и `update`, метод вернет количество 
подходящих под условие:

    use Illuminate\Support\Facades\DB;

    $deleted = DB::delete('delete from users');

<a name="running-a-general-statement"></a>
#### Выполнение общего запроса

Некоторые операторы базы данных не возвращают никакого значения. Для этих типов операций вы можете использовать 
метод `statement` фасада `DB`:

    DB::statement('drop table users');

<a name="running-an-unprepared-statement"></a>
#### Выполнение неподготовленного запроса

По желанию может потребоваться выполнить запрос SQL без привязки каких-либо значений. Для этого вы можете использовать
метод `unprepared` фасада `DB`:

    DB::unprepared('update users set votes = 100 where name = "Dries"');

> [!WARNING]
> Поскольку неподготовленные запросы не связывают параметры, они могут быть уязвимы для SQL-инъекций. Вы никогда не 
> должны пропускать в неподготовленное выражение значения, управляемые пользователем.

<a name="implicit-commits-in-transactions"></a>
#### Неявные фиксации (implicit commit)

При использовании в транзакциях методов `statement` и `unprepared` фасада `DB` вы должны быть осторожны, чтобы избежать 
операторов, которые вызывают [неявные фиксации](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html). Эти операторы заставят ядро базы данных косвенно зафиксировать 
всю транзакцию, в результате чего Laravel не будет знать об уровне транзакции базы данных. Примером такого оператора 
является создание таблицы базы данных:

    DB::unprepared('create table a (col varchar(1) null)');

Пожалуйста, обратитесь к руководству по MySQL для ознакомления со [списком всех операторов](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html), которые выполняют 
неявные фиксации.

<a name="using-multiple-database-connections"></a>
### Использование нескольких соединений к базе данных

Если ваше приложение определяет несколько соединений в конфигурационном файле `config/database.php`, то вы можете получить 
доступ к каждому соединению с помощью метода `connection` фасада `DB`. Имя соединения, передаваемое методу `connection`, 
должно соответствовать одному из подключений, перечисленных в вашем конфигурационном файле `config/database.php`, 
включая переопределенные с помощью глобального помощника `config` во время выполнения скрипта:

    use Illuminate\Support\Facades\DB;

    $users = DB::connection('sqlite')->select(/* ... */);

Вы можете получить доступ к сырому, базовому экземпляру PDO текущего соединения, используя метод `getPdo` 
экземпляра соединения:

    $pdo = DB::connection()->getPdo();

<a name="listening-for-query-events"></a>
### Прослушивание событий запроса

По желанию можно указать замыкание, которое будет вызываться для каждого SQL-запроса, выполняемого вашим приложением, 
используя метод `listen` фасада `DB`. Этот метод может быть полезен для логирования запросов или их отладки. Вы можете 
зарегистрировать замыкание слушателя запросов в методе `boot` [сервис-провайдера](/docs/{{version}}/providers):

    <?php

    namespace App\Providers;

    use Illuminate\Database\Events\QueryExecuted;
    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\ServiceProvider;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Регистрация сервисов приложения.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Загрузка сервисов приложения.
         */
        public function boot(): void
        {
            DB::listen(function (QueryExecuted $query) {
                // $query->sql;
                // $query->bindings;
                // $query->time;
            });
        }
    }


<a name="monitoring-cumulative-query-time"></a>
### Мониторинг общего времени выполнения запроса

Одной из обычных узких точек производительности современных веб-приложений является время, которое они затрачивают на 
выполнение запросов к базе данных. К счастью, Laravel может вызвать замыкание или обратный вызов по вашему выбору, когда 
время выполнения запросов к базе данных в течение одного запроса становится слишком большим. Для начала укажите порог 
времени выполнения запроса (в миллисекундах) и замыкание для метода `whenQueryingForLongerThan`. Вы можете вызвать 
этот метод в методе `boot` [сервис-провайдера](/docs/{{version}}/providers)::


    <?php

    namespace App\Providers;

    use Illuminate\Database\Connection;
    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\ServiceProvider;
    use Illuminate\Database\Events\QueryExecuted;

    class AppServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         */
        public function register(): void
        {
            // ...
        }

        /**
         * Bootstrap any application services.
         */
        public function boot(): void
        {
            DB::whenQueryingForLongerThan(500, function (Connection $connection, QueryExecuted $event) {
                // Notify development team...
            });
        }
    }

<a name="database-transactions"></a>
## Транзакции базы данных

Вы можете использовать метод `transaction` фасада `DB`, для выполнения набора операций в транзакции базы данных. 
Если при закрытии транзакции возникает исключение, то транзакция автоматически откатывается, а исключение генерируется 
повторно. Если замыкание выполнено успешно, то транзакция будет автоматически зафиксирована. Вам не нужно беспокоиться 
о ручном откате или фиксации при использовании метода `transaction`:

    use Illuminate\Support\Facades\DB;

    DB::transaction(function () {
        DB::update('update users set votes = 1');

        DB::delete('delete from posts');
    });

<a name="handling-deadlocks"></a>
#### Обработка взаимных блокировок (deadlock)

Метод `transaction` принимает необязательный второй аргумент, который определяет, сколько раз транзакция должна быть 
повторена, если произошла взаимная блокировка. Как только эти попытки будут исчерпаны, будет выброшено исключение:

    use Illuminate\Support\Facades\DB;

    DB::transaction(function () {
        DB::update('update users set votes = 1');

        DB::delete('delete from posts');
    }, 5);

<a name="manually-using-transactions"></a>
#### Использование транзакций вручную

Если вы хотите вручную начать транзакцию и иметь полный контроль над откатами и фиксациями, 
то вы можете использовать метод `beginTransaction` фасада `DB`:

    use Illuminate\Support\Facades\DB;

    DB::beginTransaction();

Вы можете откатить транзакцию с помощью метода `rollBack`:

    DB::rollBack();

Наконец, вы можете зафиксировать транзакцию с помощью метода `commit`:

    DB::commit();

> [!NOTE]
> Методы транзакций фасада `DB` контролируют транзакции как для [построителя запросов](/docs/{{version}}/queries), 
> так и для [Eloquent ORM](/docs/{{version}}/eloquent).

<a name="connecting-to-the-database-cli"></a>
## Подключение к базе данных с помощью интерфейса командной строки Artisan

Если вы хотите подключиться к своей базе данных с помощью командной строки, то вы можете использовать 
команду `db` Artisan:

```shell
php artisan db
```

При необходимости, вы можете указать имя соединения для подключения к базе данных, не являющееся соединением по умолчанию:

```shell
php artisan db mysql
```

<a name="inspecting-your-databases"></a>
## Инспектирование базы данных

С помощью команд Artisan `db:show` и `db:table` вы можете получить ценную информацию о вашей базе данных и ее связанных 
таблицах. Для просмотра информации о вашей базы данных, включая ее размер, тип, количество открытых соединений и сводку по ее 
таблицам, вы можете использовать команду `db:show`:

```shell
php artisan db:show
```

Вы можете указать, какое соединение с базой данных следует использовать, передав имя соединения с помощью опции `--database`:
```shell
php artisan db:show --database=pgsql
```

Если вы хотите включить количество строк в таблицах и подробности о представлениях базы данных в выводе команды, 
вы можете указать соответственно опции `--counts` и -`-views`. На больших базах данных получение количества строк и 
сведений о представлениях может занять много времени:

```shell
php artisan db:show --counts --views
```

<a name="table-overview"></a>
#### Обзор таблиц

Если вы хотите получить сводку по отдельной таблицы в вашей базе данных, вы можете выполнить команду Artisan `db:table`. 
Эта команда предоставляет общий обзор таблицы базы данных, включая ее столбцы, типы, атрибуты, ключи и индексы:

```shell
php artisan db:table users
```

<a name="monitoring-your-databases"></a>
## Мониторинг баз данных

Используя команду Artisan `db:monitor`, вы можете поручить Laravel отправить `Illuminate\Database\Events\DatabaseBusy`, 
если ваша база данных управляет большим количеством открытых соединений, чем задано.

Для начала вам следует запланировать выполнение команды `db:monitor` [каждую минуту](/docs/{{version}}/scheduling). Команда принимает имена 
конфигураций подключений к базе данных, которые вы хотите мониторить, а также максимальное количество открытых соединений, 
которые допустимы до отправки события:

```shell
php artisan db:monitor --databases=mysql,pgsql --max=100
```

Одного планирования этой команды недостаточно для отправки уведомления о количестве открытых соединений. 
Когда команда обнаруживает базу данных с количеством открытых соединений, превышающим ваш порог, будет отправлено 
событие `DatabaseBusy`. Вы должны прослушивать это событие в файле `EventServiceProvider` вашего приложения, 
чтобы отправить уведомление вам или вашей команде разработки:

```php
use App\Notifications\DatabaseApproachingMaxConnections;
use Illuminate\Database\Events\DatabaseBusy;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Notification;

/**
 * Зарегистрируйте любые другие события для вашего приложения.
 */
public function boot(): void
{
    Event::listen(function (DatabaseBusy $event) {
        Notification::route('mail', 'dev@example.com')
                ->notify(new DatabaseApproachingMaxConnections(
                    $event->connectionName,
                    $event->connections
                ));
    });
}
```
