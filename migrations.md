---
git: dde9f07b782ba5a3ff93e34471bf2b1a581b0a76
---

# База данных · Миграции

<a name="introduction"></a>
## Введение

Миграции похожи на контроль версий для вашей базы данных, позволяют вашей команде определять схемы базы данных приложения и совместно использовать их определение. Если вам когда-либо приходилось указывать товарищу по команде вручную добавить столбец в его схему локальной базы данных после применения изменений в системе управления версиями, то вы столкнулись с проблемой, которую решает миграция базы данных.

[Фасад](/docs/{{version}}/facades) `Schema` обеспечивает независимую от базы данных поддержку для создания и управления таблицами во всех поддерживаемых Laravel системах баз данных. В обычной ситуации, этот фасад используется для создания и изменения таблиц / столбцов базы данных во время миграции.

<a name="generating-migrations"></a>
## Генерация миграций

Чтобы сгенерировать новую миграцию базы данных, используйте команду `make:migration` [Artisan](artisan). Эта команда поместит новый класс миграции в каталог `database/migrations` вашего приложения. Каждое имя файла миграции содержит временную метку, которая позволяет Laravel определять порядок применения миграций:

```shell
php artisan make:migration create_flights_table
```

Laravel будет использовать имя миграции, чтобы попытаться угадать имя таблицы и будет ли миграция создавать новую таблицу. Если Laravel может определить имя таблицы по имени миграции, то сгенерированный файл миграции будет предварительно заполнен указанной таблицей. В противном случае вы можете просто вручную указать таблицу в файле миграции.

Если вы хотите указать собственный путь для сгенерированной миграции, вы можете использовать параметр `--path` при выполнении команды `make:migration`. Указанный путь должен быть относительно базового пути вашего приложения.

> [!NOTE]  
> Заготовки (stubs) миграции можно настроить с помощью [публикации заготовок](artisan#stub-customization).

<a name="squashing-migrations"></a>
### Сжатие миграций

По мере создания приложения вы можете со временем накапливать все больше и больше миграций. Это может привести к тому, что ваш каталог `database/migrations` станет раздутым из-за потенциально сотен миграций. Если хотите, то можете «сжать» свои миграции в один файл SQL. Для начала выполните команду `schema:dump`:

```shell
php artisan schema:dump

# Выгрузить текущую схему БД и удалить все существующие миграции ...
php artisan schema:dump --prune
```

В результате выполнения этой команды Laravel запишет дамп базы данных в каталог `database/schema` вашего приложения. Теперь, при запуске миграции базы данных, Laravel сначала выполнит SQL-операторы дампа (если никакие другие миграции не выполнялись). Затем Laravel выполнит все оставшиеся миграции, которые не были включены в дамп схемы БД.


Если ваши тесты приложения используют другое подключение к базе данных, чем то, которое вы обычно используете во время локальной разработки, убедитесь, что вы создали файл схемы с использованием этого подключения к базе данных, чтобы ваши тесты могли создать базу данных. Вы можете сделать это после создания файла схемы для базы данных, которую обычно используете во время локальной разработки:

```shell
php artisan schema:dump
php artisan schema:dump --database=testing --prune
```


Вы должны передать файл схемы базы данных в систему управления версиями, чтобы другие разработчики вашей команды могли быстро воссоздать исходную структуру базы данных вашего приложения.

> [!WARNING]  
> Сжатие миграции доступно только для баз данных MariaDB, MySQL, PostgreSQL и SQLite и использует клиент командной строки базы данных.

<a name="migration-structure"></a>
## Структура миграций

Класс миграции содержит два метода: `up` и `down`. Метод `up` используется для добавления новых таблиц, столбцов или индексов в вашу базу данных, тогда как метод `down` должен отменять операции, выполняемые методом `up`.

В обоих этих методах вы можете использовать построитель схем Laravel для выразительного создания и изменения таблиц. Чтобы узнать обо всех методах, доступных построителю `Schema`, [просмотрите его документацию](#creating-tables). Например, следующая миграция создает таблицу `flights`:

    <?php

    use Illuminate\Database\Migrations\Migration;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    return new class extends Migration
    {
        /**
         * Запустить миграцию.
         */
        public function up(): void
        {
            Schema::create('flights', function (Blueprint $table) {
                $table->id();
                $table->string('name');
                $table->string('airline');
                $table->timestamps();
            });
        }

        /**
         * Обратить миграции.
         */
        public function down(): void
        {
            Schema::drop('flights');
        }
    }

<a name="setting-the-migration-connection"></a>
#### Указание соединения миграции

Если ваша миграция будет использовать соединение с базой данных, отличное от соединения с базой данных по умолчанию, то необходимо установить свойство `$connection` миграции:

    /**
     * Соединение с БД, которое должно использоваться миграцией.
     *
     * @var string
     */
    protected $connection = 'pgsql';

    /**
     * Запустить миграцию.
     */
    public function up(): void
    {
        // ...
    }

<a name="running-migrations"></a>
## Запуск миграций

Чтобы запустить все незавершенные миграции, выполните команду `migrate` Artisan:

```shell
php artisan migrate
```

Если вы хотите узнать, какие миграции уже выполнены, то вы можете использовать команду `migrate:status` Artisan:

```shell
php artisan migrate:status
```

Если вы хотите посмотреть SQL-запросы, которые будут выполнены миграциями, но при этом не запускать их фактически, вы можете добавить флаг `--pretend` к команде `migrate`:

```shell
php artisan migrate --pretend
```

#### Изолированное выполнение миграций

Если вы развертываете свое приложение на нескольких серверах и выполняете миграции в рамках процесса развертывания, вероятно, вам не захочется, чтобы два сервера попытались выполнить миграцию базы данных одновременно. Для избежания этого вы можете использовать опцию `isolated` при вызове команды `migrate`.

Когда указана опция `isolated`, Laravel получит атомарный блокировщик с использованием драйвера кэша вашего приложения перед попыткой запуска миграций. Все другие попытки выполнить команду `migrate` в то время, как этот блокировщик удерживается, не будут выполнены; однако команда все равно завершит свое выполнение с успешным кодом статуса:

```shell
php artisan migrate --isolated
```

> [!WARNING]
> Для использования этой функции ваше приложение должно использовать драйвер кэша `memcached`, `redis`, `dynamodb`, `database`, `file` или `array` как драйвер кэша по умолчанию. Кроме того, все серверы должны общаться с одним и тем же центральным сервером кэша.

<a name="forcing-migrations-to-run-in-production"></a>
#### Принудительный запуск миграции в рабочем окружении

Некоторые операции миграции являются деструктивными, что означает, что они могут привести к потере данных. Чтобы защитить вас от запуска этих команд для вашей производственной базы данных, от вас потребуется подтверждение перед выполнением команд. Чтобы команды запускались без подтверждения, используйте флаг `--force`:

```shell
php artisan migrate --force
```

<a name="rolling-back-migrations"></a>
### Откат миграций

Чтобы откатить последнюю операцию миграции, вы можете использовать команду `rollback` Artisan. Эта команда откатывает последний «пакет» миграций, который может включать несколько файлов миграции:

```shell
php artisan migrate:rollback
```

Вы можете откатить ограниченное количество миграций, указав параметр `step` для команды `rollback`. Например, следующая команда откатит последние пять миграций:

```shell
php artisan migrate:rollback --step=5
````

Вы можете откатить определенную "партию" миграций, указав опцию `batch` команде `rollback`, где значение опции `batch` соответствует значению партии в таблице `migrations` вашего приложения. Например, следующая команда откатит все миграции в партии третьей:

```shell
php artisan migrate:rollback --batch=3
```

Если вы хотите посмотреть SQL-запросы, которые будут выполнены миграциями без их фактического выполнения, вы можете добавить флаг `--pretend` к команде `migrate:rollback`:

```shell
php artisan migrate:rollback --pretend
```

Команда `migrate:reset` откатит все миграции вашего приложения:

```shell
php artisan migrate:reset
```

<a name="roll-back-migrate-using-a-single-command"></a>
#### Откат и миграция с помощью одной команды

Команда `migrate:refresh` откатит все ваши миграции, а затем выполнит команду `migrate`. Эта команда эффективно воссоздает всю вашу базу данных:

```shell
php artisan migrate:refresh

// Обновляем базу данных и запускаем все наполнители базы данных ...
php artisan migrate:refresh --seed
```

Вы можете откатить и повторно запустить ограниченное количество миграций, указав параметр `step` для команды `refresh`. Например, следующая команда откатит и повторно запустит последние пять миграций:

```shell
php artisan migrate:refresh --step=5
```

<a name="drop-all-tables-migrate"></a>
#### Удаление всех таблиц с последующей миграцией

Команда `migrate:fresh` удалит все таблицы из базы данных, а затем выполнит команду `migrate`:

```shell
php artisan migrate:fresh

php artisan migrate:fresh --seed
```

По умолчанию команда `migrate:fresh` удаляет только таблицы из соединения с базой данных по умолчанию.
Однако вы можете использовать опцию `--database`, чтобы указать соединение с базой данных, которое следует использовать.
Имя соединения с базой данных должно соответствовать имени, определенному в [файле конфигурации базы данных](/docs/{{version}}/configuration) вашего приложения:

```shell
php artisan migrate:fresh --database=admin
```

> [!WARNING] 
> Команда `migrate:fresh` удалит все таблицы базы данных независимо от их префикса. Эту команду следует использовать с осторожностью при разработке в базе данных, которая используется совместно с другими приложениями.

<a name="tables"></a>
## Таблицы

<a name="creating-tables"></a>
### Создание таблиц

Чтобы создать новую таблицу базы данных, используйте метод `create` фасада `Schema`. Метод `create` принимает два аргумента: первый – это имя таблицы, а второй – замыкание, которое получает объект `Blueprint`, используемый для определения новой таблицы:

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::create('users', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->string('email');
        $table->timestamps();
    });

При создании таблицы вы можете использовать любой из [методов столбцов](#creating-columns) построителя схемы для определения столбцов таблицы.

<a name="determining-table-column-existence"></a>
#### Определение наличия таблицы / столбца

Вы можете определить наличие таблицы, столбца или индекса с помощью методов `hasTable`, `hasColumn` и `hasIndex`, соответственно:

    if (Schema::hasTable('users')) {
        // Таблица `users` существует ...
    }

    if (Schema::hasColumn('users', 'email')) {
        // Таблица `users` существует и содержит столбец `email` ...
    }

    if (Schema::hasIndex('users', ['email'], 'unique')) {
        // Таблица `users` существует и имеет уникальный индекс в столбце `email`...
    }

<a name="database-connection-table-options"></a>
#### Соединение с базой данных и параметры таблицы

Если вы хотите выполнить операцию схемы с подключением, которое не является подключением к базе данных по умолчанию для вашего приложения, используйте метод `connection`:

    Schema::connection('sqlite')->create('users', function (Blueprint $table) {
        $table->id();
    });

Кроме того, некоторые другие свойства и методы могут использоваться для определения других аспектов создания таблицы. Свойство `engine` используется для указания механизма хранения таблицы при использовании MariaDB или MySQL:

    Schema::create('users', function (Blueprint $table) {
        $table->engine('InnoDB');

        // ...
    });

Свойства `charset` и `collation` могут использоваться для указания набора символов и сопоставления для создаваемой таблицы при использовании MariaDB или MySQL:

    Schema::create('users', function (Blueprint $table) {
        $table->charset('utf8mb4');
        $table->collation('utf8mb4_unicode_ci');

        // ...
    });

Метод `temporary` используется, чтобы указать, что таблица должна быть «временной». Временные таблицы видны только текущему сеансу соединения базы данных и автоматически удаляются при закрытии соединения:

    Schema::create('calculations', function (Blueprint $table) {
        $table->temporary();

        // ...
    });

Если вы хотите добавить "комментарий" к таблице базы данных, вы можете вызвать метод `comment` на экземпляре таблицы. Комментарии к таблицам поддерживаются только в MariaDB, MySQL и Postgres:

```php
Schema::create('calculations', function (Blueprint $table) {
    $table->comment('Business calculations');

    // ...
});
```

Этот код позволит вам добавить комментарий "Business calculations" к таблице "calculations" в вашей базе данных. Это может быть полезно для документации и описания цели таблицы.

<a name="updating-tables"></a>
### Обновление таблиц

Метод `table` фасада `Schema` используется для обновления существующих таблиц. Подобно методу `create`, метод `table` принимает два аргумента: имя таблицы и замыкание, которое получает экземпляр `Blueprint`, используемый для добавления столбцов или индексов в таблицу:

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes');
    });

<a name="renaming-and-dropping-tables"></a>
### Переименование / удаление таблиц

Чтобы переименовать существующую таблицу базы данных, используйте метод `rename`:

    use Illuminate\Support\Facades\Schema;

    Schema::rename($from, $to);

Чтобы удалить существующую таблицу, вы можете использовать методы `drop` или `dropIfExists`:

    Schema::drop('users');

    Schema::dropIfExists('users');

<a name="renaming-tables-with-foreign-keys"></a>
#### Переименование таблиц с внешними ключами

Перед переименованием таблицы вы должны убедиться, что любые ограничения внешнего ключа в таблице имеют явное имя в ваших файлах миграции, вместо того, чтобы позволять Laravel назначать имя на основе соглашения. В противном случае имя ограничения внешнего ключа будет ссылаться на имя старой таблицы.

<a name="columns"></a>
## Столбцы

<a name="creating-columns"></a>
### Создание столбцов

Метод `table` фасада `Schema` используется для обновления существующих таблиц. Как и метод `create`, метод `table` принимает два аргумента: имя таблицы и замыкание, которое получает экземпляр `Illuminate\Database\Schema\Blueprint`, используемый для добавления столбцов в таблицу:

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->integer('votes');
    });

<a name="available-column-types"></a>
### Доступные типы столбцов

Построитель схем Blueprint предлагает множество методов, соответствующих различным типам столбцов, которые вы можете добавить в таблицы базы данных. Все доступные методы перечислены в таблице ниже:

<div class="docs-column-list" markdown="1">

- [bigIncrements](#column-method-bigIncrements)
- [bigInteger](#column-method-bigInteger)
- [binary](#column-method-binary)
- [boolean](#column-method-boolean)
- [char](#column-method-char)
- [dateTimeTz](#column-method-dateTimeTz)
- [dateTime](#column-method-dateTime)
- [date](#column-method-date)
- [decimal](#column-method-decimal)
- [double](#column-method-double)
- [enum](#column-method-enum)
- [float](#column-method-float)
- [foreignId](#column-method-foreignId)
- [foreignIdFor](#column-method-foreignIdFor)
- [foreignUlid](#column-method-foreignUlid)
- [foreignUuid](#column-method-foreignUuid)
- [geography](#column-method-geography)
- [geometry](#column-method-geometry)
- [id](#column-method-id)
- [increments](#column-method-increments)
- [integer](#column-method-integer)
- [ipAddress](#column-method-ipAddress)
- [json](#column-method-json)
- [jsonb](#column-method-jsonb)
- [longText](#column-method-longText)
- [macAddress](#column-method-macAddress)
- [mediumIncrements](#column-method-mediumIncrements)
- [mediumInteger](#column-method-mediumInteger)
- [mediumText](#column-method-mediumText)
- [morphs](#column-method-morphs)
- [nullableMorphs](#column-method-nullableMorphs)
- [nullableTimestamps](#column-method-nullableTimestamps)
- [nullableUlidMorphs](#column-method-nullableUlidMorphs)
- [nullableUuidMorphs](#column-method-nullableUuidMorphs)
- [rememberToken](#column-method-rememberToken)
- [set](#column-method-set)
- [smallIncrements](#column-method-smallIncrements)
- [smallInteger](#column-method-smallInteger)
- [softDeletesTz](#column-method-softDeletesTz)
- [softDeletes](#column-method-softDeletes)
- [string](#column-method-string)
- [text](#column-method-text)
- [timeTz](#column-method-timeTz)
- [time](#column-method-time)
- [timestampTz](#column-method-timestampTz)
- [timestamp](#column-method-timestamp)
- [timestampsTz](#column-method-timestampsTz)
- [timestamps](#column-method-timestamps)
- [tinyIncrements](#column-method-tinyIncrements)
- [tinyInteger](#column-method-tinyInteger)
- [tinyText](#column-method-tinyText)
- [unsignedBigInteger](#column-method-unsignedBigInteger)
- [unsignedInteger](#column-method-unsignedInteger)
- [unsignedMediumInteger](#column-method-unsignedMediumInteger)
- [unsignedSmallInteger](#column-method-unsignedSmallInteger)
- [unsignedTinyInteger](#column-method-unsignedTinyInteger)
- [ulidMorphs](#column-method-ulidMorphs)
- [uuidMorphs](#column-method-uuidMorphs)
- [ulid](#column-method-ulid)
- [uuid](#column-method-uuid)
- [vector](#column-method-vector)
- [year](#column-method-year)

</div>

<a name="column-method-bigIncrements"></a>
#### `bigIncrements()`

Метод `bigIncrements` создает эквивалент автоинкрементного столбца `UNSIGNED BIGINT` (первичный ключ):

    $table->bigIncrements('id');

<a name="column-method-bigInteger"></a>
#### `bigInteger()`

Метод `bigInteger` создает эквивалент столбца `BIGINT`:

    $table->bigInteger('votes');

<a name="column-method-binary"></a>
#### `binary()`

Метод `binary` создает эквивалент столбца `BLOB`:

    $table->binary('photo');

При использовании MySQL, MariaDB или SQL Server вы можете передать аргументы `length` и `fixed` для создания эквивалентного столбца `VARBINARY` или `BINARY`:

    $table->binary('data', length: 16); // VARBINARY(16)

    $table->binary('data', length: 16, fixed: true); // BINARY(16)

<a name="column-method-boolean"></a>
#### `boolean()`

Метод `boolean` создает эквивалент столбца `BOOLEAN`:

    $table->boolean('confirmed');

<a name="column-method-char"></a>
#### `char()`

Метод `char` создает эквивалент столбца `CHAR` указанной длины:

    $table->char('name', length: 100);

<a name="column-method-dateTimeTz"></a>
#### `dateTimeTz()`

Метод `dateTimeTz` создает эквивалент столбца `DATETIME` (с часовым поясом) с необязательной точностью до долей секунды:

    $table->dateTimeTz('created_at', precision: 0);

<a name="column-method-dateTime"></a>
#### `dateTime()`

Метод `dateTime` создает эквивалент столбца `DATETIME` с необязательной точностью до долей секунды:

    $table->dateTime('created_at', precision: 0);

<a name="column-method-date"></a>
#### `date()`

Метод `date` создает эквивалент столбца `DATE`:

    $table->date('created_at');

<a name="column-method-decimal"></a>
#### `decimal()`

Метод `decimal` создает эквивалент столбца `DECIMAL` с точностью (общее количество цифр) и масштабом (десятичные цифры):

    $table->decimal('amount', total: 8, places: 2);

<a name="column-method-double"></a>
#### `double()`

Метод `double` создает эквивалент столбца `DOUBLE`:

    $table->double('amount');

<a name="column-method-enum"></a>
#### `enum()`

Метод `enum` создает эквивалент столбца `ENUM` с указанием допустимых значений:

    $table->enum('difficulty', ['easy', 'hard']);

<a name="column-method-float"></a>
#### `float()`

Метод `float` создает эквивалент столбца `FLOAT` с заданной точностью:

    $table->float('amount', precision: 53);

<a name="column-method-foreignId"></a>
#### `foreignId()`

Метод `foreignId` создает эквивалент столбца `UNSIGNED BIGINT`:

    $table->foreignId('user_id');

<a name="column-method-foreignIdFor"></a>
#### `foreignIdFor()`

Метод `foreignIdFor` добавляет столбец с именем `{column}_id`, эквивалентный для заданного класса модели. Тип столбца будет `UNSIGNED BIGINT`, `CHAR(36)` или `CHAR(26)`, в зависимости от типа ключа модели:

    $table->foreignIdFor(User::class);

<a name="column-method-foreignUlid"></a>
#### `foreignUlid()`

Метод `foreignUlid` создает столбец, эквивалентный `ULID`:

    $table->foreignUlid('user_id');

<a name="column-method-foreignUuid"></a>
#### `foreignUuid()`

Метод `foreignUuid` создает эквивалент столбца `UUID`:

    $table->foreignUuid('user_id');

<a name="column-method-geography"></a>
#### `geography()`

Метод `geography` создает эквивалент столбца `GEOGRAPHY` с заданным пространственным типом и SRID (идентификатором пространственной системы отсчета):

    $table->geography('coordinates', subtype: 'point', srid: 4326);

> [!NOTE]  
> Поддержка пространственных типов зависит от драйвера вашей базы данных. Пожалуйста, обратитесь к документации вашей базы данных. Если ваше приложение использует базу данных PostgreSQL, вам необходимо установить расширение [PostGIS](https://postgis.net), прежде чем можно будет использовать метод `geography`.

<a name="column-method-geometry"></a>
#### `geometry()`

Метод `geometry` создает эквивалент столбца `GEOMETRY` с заданным пространственным типом и SRID (идентификатором пространственной системы отсчета):

    $table->geometry('positions', subtype: 'point', srid: 0);

> [!NOTE]  
> Поддержка пространственных типов зависит от драйвера вашей базы данных. Пожалуйста, обратитесь к документации вашей базы данных. Если ваше приложение использует базу данных PostgreSQL, вам необходимо установить расширение [PostGIS](https://postgis.net), прежде чем можно будет использовать метод `geometry`.

<a name="column-method-id"></a>
#### `id()`

Метод `id` является псевдонимом метода `bigIncrements`. По умолчанию метод создает столбец `id`; однако, вы можете передать имя столбца, если хотите присвоить столбцу другое имя:

    $table->id();

<a name="column-method-increments"></a>
#### `increments()`

Метод `increments` создает эквивалент автоинкрементного столбца `UNSIGNED INTEGER` в качестве первичного ключа:

    $table->increments('id');

<a name="column-method-integer"></a>
#### `integer()`

Метод `integer` создает эквивалент столбца `INTEGER`:

    $table->integer('votes');

<a name="column-method-ipAddress"></a>
#### `ipAddress()`

Метод `ipAddress` создает эквивалент столбца `VARCHAR`:

    $table->ipAddress('visitor');

При использовании PostgreSQL будет создан столбец `INET`.

<a name="column-method-json"></a>
#### `json()`

Метод `json` создает эквивалент столбца `JSON`:

    $table->json('options');

<a name="column-method-jsonb"></a>
#### `jsonb()`

Метод `jsonb` создает эквивалент столбца `JSONB`:

    $table->jsonb('options');

<a name="column-method-longText"></a>
#### `longText()`

Метод `longText` создает эквивалент столбца `LONGTEXT`:

    $table->longText('description');

При использовании MySQL или MariaDB вы можете применить к столбцу `binary` набор символов, чтобы создать эквивалентный столбец `LONGBLOB`:

    $table->longText('data')->charset('binary'); // LONGBLOB

<a name="column-method-macAddress"></a>
#### `macAddress()`

Метод `macAddress` создает столбец, предназначенный для хранения MAC-адреса. Некоторые системы баз данных, такие как PostgreSQL, имеют специальный тип столбца для этого типа данных. Другие системы баз данных будут использовать столбец строкового эквивалента:

    $table->macAddress('device');

<a name="column-method-mediumIncrements"></a>
#### `mediumIncrements()`

Метод `mediumIncrements` создает эквивалент автоинкрементного столбца `UNSIGNED MEDIUMINT` в качестве первичного ключа:

    $table->mediumIncrements('id');

<a name="column-method-mediumInteger"></a>
#### `mediumInteger()`

Метод `mediumInteger` создает эквивалент столбца `MEDIUMINT`:

    $table->mediumInteger('votes');

<a name="column-method-mediumText"></a>
#### `mediumText()`

Метод `mediumText` создает эквивалент столбца `MEDIUMTEXT`:

    $table->mediumText('description');

При использовании MySQL или MariaDB вы можете применить к столбцу `binary` набор символов, чтобы создать эквивалентный столбец `MEDIUMBLOB`:

    $table->mediumText('data')->charset('binary'); // MEDIUMBLOB

<a name="column-method-morphs"></a>
#### `morphs()`

Метод `morphs` - это удобный метод, который добавляет эквивалент столбца `{column}_id` и столбца `{column}_type` с типом данных `VARCHAR`. Тип данных столбца `{column}_id` будет `UNSIGNED BIGINT`, `CHAR(36)` или `CHAR(26)`, в зависимости от типа ключа модели.

Этот метод предназначен для использования при определении столбцов, необходимых для полиморфного [отношения Eloquent](/docs/{{version}}/eloquent-relationships). В следующем примере будут созданы столбцы `taggable_id` и `taggable_type`:

    $table->morphs('taggable');

<a name="column-method-nullableTimestamps"></a>
#### `nullableTimestamps()`

Метод `nullableTimestamps` является псевдонимом метода [`timestamps`](#column-method-timestamps):

    $table->nullableTimestamps(precision: 0);

<a name="column-method-nullableMorphs"></a>
#### `nullableMorphs()`

Метод аналогичен методу [`morphs`](#column-method-morphs); тем не менее, создаваемый столбец будет иметь значение NULL:

    $table->nullableMorphs('taggable');

<a name="column-method-nullableUlidMorphs"></a>
#### `nullableUlidMorphs()`

Этот метод аналогичен методу [ulidMorphs](#column-method-ulidMorphs); однако создаваемые столбцы будут "nullable" (допускающими значение null):

```php
$table->nullableUlidMorphs('taggable');
```

<a name="column-method-nullableUuidMorphs"></a>
#### `nullableUuidMorphs()`

Метод аналогичен методу [`uuidMorphs`](#column-method-uuidMorphs); тем не менее, создаваемый столбец будет иметь значение NULL:

    $table->nullableUuidMorphs('taggable');

<a name="column-method-rememberToken"></a>
#### `rememberToken()`

Метод `rememberToken` создает NULL-эквивалент столбца `VARCHAR(100)`, предназначенный для хранения текущего [токена аутентификации](authentication#remembering-users):

    $table->rememberToken();

<a name="column-method-set"></a>
#### `set()`

Метод `set` создает эквивалент столбца `SET` с заданным списком допустимых значений:

    $table->set('flavors', ['strawberry', 'vanilla']);

<a name="column-method-smallIncrements"></a>
#### `smallIncrements()`

Метод `smallIncrements` создает эквивалент автоинкрементного столбца `UNSIGNED SMALLINT` в качестве первичного ключа:

    $table->smallIncrements('id');

<a name="column-method-smallInteger"></a>
#### `smallInteger()`

Метод `smallInteger` создает эквивалент столбца `SMALLINT`:

    $table->smallInteger('votes');

<a name="column-method-softDeletesTz"></a>
#### `softDeletesTz()`

Метод `softDeletesTz` добавляет NULL-эквивалент столбца `TIMESTAMP` (с часовым поясом) с необязательной точностью до долей секунды. Этот столбец предназначен для хранения временной метки `deleted_at`, необходимой для функции «программного удаления» Eloquent:

    $table->softDeletesTz('deleted_at', precision: 0);

<a name="column-method-softDeletes"></a>
#### `softDeletes()`

Метод `softDeletes` добавляет NULL-эквивалент столбца `TIMESTAMP` с необязательной точностью до долей секунды. Этот столбец предназначен для хранения временной метки `deleted_at`, необходимой для функции «программного удаления» Eloquent:

    $table->softDeletes('deleted_at', precision: 0);

<a name="column-method-string"></a>
#### `string()`

Метод `string` создает эквивалент столбца `VARCHAR` указанной длины:

    $table->string('name', length: 100);

<a name="column-method-text"></a>
#### `text()`

Метод `text` создает эквивалент столбца `TEXT`:

    $table->text('description');

При использовании MySQL или MariaDB вы можете применить к столбцу `binary` набор символов, чтобы создать эквивалент столбца `BLOB`:

    $table->text('data')->charset('binary'); // BLOB

<a name="column-method-timeTz"></a>
#### `timeTz()`

Метод `timeTz` создает эквивалент столбца `TIME` (с часовым поясом) с необязательной точностью до долей секунды:

    $table->timeTz('sunrise', precision: 0);

<a name="column-method-time"></a>
#### `time()`

Метод `time` создает эквивалент столбца `TIME` с необязательной точностью до долей секунды:

    $table->time('sunrise', precision: 0);

<a name="column-method-timestampTz"></a>
#### `timestampTz()`

Метод `timestampTz` создает эквивалент столбца `TIMESTAMP` (с часовым поясом) с необязательной точностью до долей секунды:

    $table->timestampTz('added_at', precision: 0);

<a name="column-method-timestamp"></a>
#### `timestamp()`

Метод `timestamp` создает эквивалент столбца `TIMESTAMP` с необязательной точностью до долей секунды:

    $table->timestamp('added_at', precision: 0);

<a name="column-method-timestampsTz"></a>
#### `timestampsTz()`

Метод `timestampsTz` создает столбцы `created_at` и `updated_at`, эквивалентные `TIMESTAMP` (с часовым поясом) с необязательной точностью до долей секунды:

    $table->timestampsTz(precision: 0);

<a name="column-method-timestamps"></a>
#### `timestamps()`

Метод `timestamps` method creates `created_at` and `updated_at` `TIMESTAMP` с необязательной точностью до долей секунды:

    $table->timestamps(precision: 0);

<a name="column-method-tinyIncrements"></a>
#### `tinyIncrements()`

Метод `tinyIncrements` создает эквивалент автоинкрементного столбца `UNSIGNED TINYINT` в качестве первичного ключа:

    $table->tinyIncrements('id');

<a name="column-method-tinyInteger"></a>
#### `tinyInteger()`

Метод `tinyInteger` создает эквивалент столбца `TINYINT`:

    $table->tinyInteger('votes');

<a name="column-method-tinyText"></a>
#### `tinyText()`

Метод `tinyText` создаёт эквивалент столбца `TINYTEXT`:

    $table->tinyText('notes');    

When utilizing MySQL or MariaDB, you may apply a `binary` character set to the column in order to create a `TINYBLOB` equivalent column:
При использовании MySQL или MariaDB вы можете применить к столбцу `binary` набор символов, чтобы создать эквивалентный столбец `TINYBLOB`:

    $table->tinyText('data')->charset('binary'); // TINYBLOB

<a name="column-method-unsignedBigInteger"></a>
#### `unsignedBigInteger()`

Метод `unsignedBigInteger` создает эквивалент столбца `UNSIGNED BIGINT`:

    $table->unsignedBigInteger('votes');

<a name="column-method-unsignedInteger"></a>
#### `unsignedInteger()`

Метод `unsignedInteger` создает эквивалент столбца `UNSIGNED INTEGER`:

    $table->unsignedInteger('votes');

<a name="column-method-unsignedMediumInteger"></a>
#### `unsignedMediumInteger()`

Метод `unsignedMediumInteger` создает эквивалент столбца `UNSIGNED MEDIUMINT`:

    $table->unsignedMediumInteger('votes');

<a name="column-method-unsignedSmallInteger"></a>
#### `unsignedSmallInteger()`

Метод `unsignedSmallInteger` создает эквивалент столбца `UNSIGNED SMALLINT`:

    $table->unsignedSmallInteger('votes');

<a name="column-method-unsignedTinyInteger"></a>
#### `unsignedTinyInteger()`

Метод `unsignedTinyInteger` создает эквивалент столбца `UNSIGNED TINYINT`:

    $table->unsignedTinyInteger('votes');

<a name="column-method-ulidMorphs"></a>
#### Метод `ulidMorphs()`

Метод `ulidMorphs` - это удобный метод, который добавляет эквивалент столбца `{column}_id` типа `CHAR(26)` и столбца `{column}_type` типа `VARCHAR`.

Этот метод предназначен для использования при определении столбцов, необходимых для полиморфных [Eloquent отношений](/docs/{{version}}/eloquent-relationships), которые используют ULID идентификаторы. В следующем примере будут созданы столбцы `taggable_id` и `taggable_type`:

    $table->ulidMorphs('taggable');

<a name="column-method-uuidMorphs"></a>
#### `uuidMorphs()`

Метод `uuidMorphs` – это удобный метод, который добавляет эквивалент столбца `CHAR(36)` (`{column}_id`) и эквивалент столбца `VARCHAR` (`{column}_type`).

Этот метод предназначен для использования при определении столбцов, необходимых для полиморфного [отношения Eloquent](/docs/{{version}}/eloquent-relationships), использующего идентификаторы UUID. В следующем примере будут созданы столбцы `taggable_id` и `taggable_type`:

    $table->uuidMorphs('taggable');

<a name="column-method-ulid"></a>
#### Метод `ulid()`

Метод `ulid` создает столбец, эквивалентный `ULID`:

```php
$table->ulid('id');
```

Метод `ulid` создает столбец, эквивалентный `ULID` и присваивает ему имя 'id'.

<a name="column-method-uuid"></a>
#### `uuid()`

Метод `uuid` создает эквивалент столбца `UUID`:

    $table->uuid('id');

<a name="column-method-vector"></a>
#### `vector()`

Метод `vector` создает эквивалент столбца `vector`:

    $table->vector('embedding', dimensions: 100);

<a name="column-method-year"></a>
#### `year()`

Метод `year` создает эквивалент столбца `YEAR`:

    $table->year('birth_year');

<a name="column-modifiers"></a>
### Модификаторы столбца

В дополнение к типам столбцов, перечисленным выше, есть несколько «модификаторов» столбцов, которые вы можете использовать при добавлении столбца в таблицу базы данных. Например, чтобы сделать столбец «допускающим значение NULL», вы можете использовать метод `nullable`:

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->string('email')->nullable();
    });

В следующей таблице представлены все доступные модификаторы столбцов. В этот список не входят [модификаторы индексов](#creating-indexes):

| Модификатор                         | Описание                                                                                                         |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `->after('column')`                 | Поместить столбец «после» другого столбца (MariaDB / MySQL).                                                     |
| `->autoIncrement()`                 | Установить столбцы INTEGER как автоинкрементные (первичный ключ).                                                |
| `->charset('utf8mb4')`              | Указать набор символов для столбца (MariaDB / MySQL).                                                            |
| `->collation('utf8mb4_unicode_ci')` | Укажить параметры сортировки для столбца.                                                                        |
| `->comment('my comment')`           | Добавить комментарий к столбцу (MariaDB / MySQL / PostgreSQL).                                                   |
| `->default($value)`                 | Указать значение «по умолчанию» для столбца.                                                                     |
| `->first()`                         | Поместить столбец «первым» в таблице (MariaDB / MySQL).                                                          |
| `->from($integer)`                  | Установить начальное значение автоинкрементного поля (MariaDB / MySQL / PostgreSQL).                             |
| `->invisible()`                     | Сделать столбец "невидимым" для запросов `SELECT *` (MariaDB / MySQL).                                           |
| `->nullable($value = true)`         | Позволить (по умолчанию) значения `NULL` для вставки в столбец.                                                  |
| `->storedAs($expression)`           | Создать сохраненный генерируемый столбец (MariaDB / MySQL / PostgreSQL / SQLite).                                |
| `->unsigned()`                      | Установить столбцы `INTEGER` как `UNSIGNED` (MariaDB / MySQL).                                                   |
| `->useCurrent()`                    | Установить столбцы `TIMESTAMP` для использования `CURRENT_TIMESTAMP` в качестве значения по умолчанию.           |
| `->useCurrentOnUpdate()`            | Установить столбцы `TIMESTAMP` для использования `CURRENT_TIMESTAMP` при обновлении записи (MariaDB / MySQL).    |
| `->virtualAs($expression)`          | Создать виртуальный генерируемый столбец (MariaDB / MySQL / SQLite).                                             |
| `->generatedAs($expression)`        | Создать столбец идентификаторов с указанными параметрами последовательности (PostgreSQL).                        |
| `->always()`                        | Определить приоритет значений последовательности над вводом для столбца идентификаторов (PostgreSQL).            |

<a name="default-expressions"></a>
#### Выражения для значений по умолчанию

Модификатор `default` принимает значение или экземпляр `Illuminate\Database\Query\Expression`. Использование экземпляра `Expression` не позволит Laravel заключить значение в кавычки и позволит вам использовать функции, специфичные для базы данных. Одна из ситуаций, когда это особенно полезно, когда вам нужно назначить значения по умолчанию для столбцов JSON:

    <?php

    use Illuminate\Support\Facades\Schema;
    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Database\Query\Expression;
    use Illuminate\Database\Migrations\Migration;

    return new class extends Migration
    {
        /**
         * Запустить миграцию.
         */
        public function up(): void
        {
            Schema::create('flights', function (Blueprint $table) {
                $table->id();
                $table->json('movies')->default(new Expression('(JSON_ARRAY())'));
                $table->timestamps();
            });
        }
    }

> [!WARNING] 
> Поддержка выражений по умолчанию зависит от вашего драйвера базы данных, версии базы данных и типа поля. См. документацию к вашей базе данных.

<a name="column-order"></a>
#### Порядок столбцов

Метод `after` добавляет набор столбцов после указанного существующего столбца в схеме базы данных MariaDB или MySQL:

    $table->after('password', function (Blueprint $table) {
        $table->string('address_line1');
        $table->string('address_line2');
        $table->string('city');
    });

<a name="modifying-columns"></a>
### Изменение столбцов

Метод `change` позволяет вам изменять тип и атрибуты существующих колонок. Например, вы можете захотеть увеличить размер колонки типа `string`. Чтобы увидеть метод `change` в действии, давайте увеличим размер колонки `name` с 25 до 50. Для этого мы просто определяем новое состояние колонки и затем вызываем метод `change`:

```php
Schema::table('users', function (Blueprint $table) {
    $table->string('name', 50)->change();
});
```

При изменении колонки вы должны явно включить все модификаторы, которые вы хотите сохранить в определении колонки - любой пропущенный атрибут будет удален. Например, чтобы сохранить атрибуты `unsigned`, `default` и `comment`, вам нужно вызывать каждый модификатор явно при изменении колонки:

```php
Schema::table('users', function (Blueprint $table) {
    $table->integer('votes')->unsigned()->default(1)->comment('мой комментарий')->change();
});
```

Метод `change` не меняет индексы столбца. Поэтому вы можете использовать модификаторы индекса, чтобы явно добавлять или удалять индекс при изменении столбца:

```php
// Add an index...
$table->bigIncrements('id')->primary()->change();

// Drop an index...
$table->char('postal_code', 10)->unique(false)->change();
```

<a name="renaming-columns"></a>
### Переименование столбцов

Для переименования столбца вы можете использовать метод `renameColumn`, предоставленный строителем схемы:

```php
Schema::table('users', function (Blueprint $table) {
    $table->renameColumn('from', 'to');
});
```

<a name="dropping-columns"></a>
### Удаление столбцов

Для удаления столбца вы можете использовать метод `dropColumn` в билдере схемы:

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn('votes');
    });

Вы можете удалить несколько столбцов из таблицы, передав массив имен столбцов методу `dropColumn`:

    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn(['votes', 'avatar', 'location']);
    });

<a name="available-command-aliases"></a>
#### Доступные псевдонимы команд

Laravel содержит несколько удобных методов, связанных с удалением общих типов столбцов. Каждый из этих методов описан в таблице ниже:

| Команда                            | Описание                                           |
| ---------------------------------- | -------------------------------------------------- |
| `$table->dropMorphs('morphable');` | Удалить столбцы `morphable_id` и `morphable_type`. |
| `$table->dropRememberToken();`     | Удалить столбец `remember_token`.                  |
| `$table->dropSoftDeletes();`       | Удалить столбец `deleted_at`.                      |
| `$table->dropSoftDeletesTz();`     | Псевдоним `dropSoftDeletes()`.                     |
| `$table->dropTimestamps();`        | Удалить столбцы `created_at` и `updated_at`.       |
| `$table->dropTimestampsTz();`      | Псевдоним `dropTimestamps()`.                      |

<a name="indexes"></a>
## Индексы

<a name="creating-indexes"></a>
### Создание индексов

Построитель схем Laravel поддерживает несколько типов индексов. В следующем примере создается новый столбец `email` и указывается, что его значения должны быть уникальными. Чтобы создать индекс, мы можем связать метод `unique` с определением столбца:

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('users', function (Blueprint $table) {
        $table->string('email')->unique();
    });

В качестве альтернативы вы можете создать индекс после определения столбца. Для этого вы должны вызвать метод `unique` построителя схемы Blueprint. Этот метод принимает имя столбца, который должен получить уникальный индекс:

    $table->unique('email');

Вы даже можете передать массив столбцов методу индекса для создания составного индекса:

    $table->index(['account_id', 'created_at']);

При создании индекса Laravel автоматически сгенерирует имя индекса на основе таблицы, имен столбцов и типа индекса, но вы можете передать второй аргумент методу, чтобы указать имя индекса самостоятельно:

    $table->unique('email', 'unique_email');

<a name="available-index-types"></a>
#### Доступные типы индексов

Построитель схем Laravel содержит методы для создания каждого типа индекса, поддерживаемого Laravel. Каждый метод индекса принимает необязательный второй аргумент для указания имени индекса. Если не указано, то имя будет производным от имен таблицы и столбцов, используемых для индекса, а также типа индекса. Все доступные методы индекса описаны в таблице ниже:

| Команда                                          | Описание                                                           |
| ------------------------------------------------ | ------------------------------------------------------------------ |
| `$table->primary('id');`                         | Добавить первичный ключ.                                           |
| `$table->primary(['id', 'parent_id']);`          | Добавить составной ключ.                                           |
| `$table->unique('email');`                       | Добавить уникальный индекс.                                        |
| `$table->index('state');`                        | Добавляет простой индекс.                                          |
| `$table->fulltext('body');`                      | Добавляет полнотекстовый индекс (MariaDB / MySQL / PostgreSQL).    |
| `$table->fulltext('body')->language('english');` | Добавляет полнотекстовый индекс для указанного языка (PostgreSQL). |
| `$table->spatialIndex('location');`              | Добавляет пространственный индекс (кроме SQLite).                  |

<a name="renaming-indexes"></a>
### Переименование индексов

Чтобы переименовать индекс, вы можете использовать метод `renameIndex` построителя схемы Blueprint. Этот метод принимает текущее имя индекса в качестве первого аргумента и желаемое имя в качестве второго аргумента:

    $table->renameIndex('from', 'to')

<a name="dropping-indexes"></a>
### Удаление индексов

Чтобы удалить индекс, вы должны указать имя индекса. По умолчанию Laravel автоматически назначает имя индекса на основе имени таблицы, имени индексированного столбца и типа индекса. Вот некоторые примеры:

| Команда                                                  | Описание                                                         |
| -------------------------------------------------------- | ---------------------------------------------------------------- |
| `$table->dropPrimary('users_id_primary');`               | Удалить первичный ключ из таблицы `users`.                       |
| `$table->dropUnique('users_email_unique');`              | Удалить уникальный индекс из таблицы `users`.                    |
| `$table->dropIndex('geo_state_index');`                  | Удалить простой индекс из таблицы `geo`.                         |
| `$table->dropFullText('posts_body_fulltext');`           | Удалить полнотекстовый индекс из таблицы `posts`.                |
| `$table->dropSpatialIndex('geo_location_spatialindex');` | Удалить пространственный индекс из таблицы `geo` (кроме SQLite). |

Если вы передадите массив столбцов в метод, удаляющий индексы, то обычное имя индекса будет сгенерировано на основе имени таблицы, столбцов и типа индекса:

    Schema::table('geo', function (Blueprint $table) {
        $table->dropIndex(['state']); // Удалить простой индекс `geo_state_index`.
    });

<a name="foreign-key-constraints"></a>
### Ограничения внешнего ключа

Laravel также поддерживает создание ограничений внешнего ключа, которые используются для обеспечения ссылочной целостности на уровне базы данных. Например, давайте определим столбец `user_id` в таблице `posts`, который ссылается на столбец `id` в таблице `users`:

    use Illuminate\Database\Schema\Blueprint;
    use Illuminate\Support\Facades\Schema;

    Schema::table('posts', function (Blueprint $table) {
        $table->unsignedBigInteger('user_id');

        $table->foreign('user_id')->references('id')->on('users');
    });

Поскольку этот синтаксис довольно подробный, Laravel предлагает дополнительные, более сжатые методы, использующие соглашения, для повышения продуктивности разработки. При использовании метода `foreignId` для создания вашего столбца приведенный выше пример можно переписать так:

    Schema::table('posts', function (Blueprint $table) {
        $table->foreignId('user_id')->constrained();
    });

Метод `foreignId` создает столбец эквивалентный `UNSIGNED BIGINT`, в то время как метод `constrained` будет использовать соглашения для определения таблицы и столбца, на которые ссылаются. Если имя вашей таблицы не соответствует соглашениям Laravel, вы можете вручную указать его в методе `constrained`. Кроме того, можно также указать имя, которое должно быть присвоено созданному индексу:

    Schema::table('posts', function (Blueprint $table) {
        $table->foreignId('user_id')->constrained(
            table: 'users', indexName: 'posts_user_id'
        );
    });

Вы также можете указать желаемое действие для свойств ограничения «при удалении» и «при обновлении»:

    $table->foreignId('user_id')
          ->constrained()
          ->onUpdate('cascade')
          ->onDelete('cascade');

Для этих действий также предусмотрен альтернативный синтаксис выражений:

| Метод                         | Описание                                                             |
| ----------------------------- | -------------------------------------------------------------------- |
| `$table->cascadeOnUpdate();`  | Обновления должны быть каскадными.                                   |
| `$table->restrictOnUpdate();` | Обновления должны быть ограничены.                                   |
| `$table->nullOnUpdate();`     | Обновления должны устанавливать значение внешнего ключа null.        |
| `$table->noActionOnUpdate();` | Никаких действий по обновлениям.                                     |
| `$table->cascadeOnDelete();`  | Удаления должны быть каскадными.                                     |
| `$table->restrictOnDelete();` | Удаления должны быть ограничены.                                     |
| `$table->nullOnDelete();`     | Для удаления следует установить значение внешнего ключа равным null. |
| `$table->noActionOnDelete();` | Предотвращает удаление, если существуют дочерние записи.             |

Любые дополнительные [модификаторы столбца](#column-modifiers) должны быть вызваны перед методом `constrained`:

    $table->foreignId('user_id')
          ->nullable()
          ->constrained();

<a name="dropping-foreign-keys"></a>
#### Удаление внешних ключей

Чтобы удалить внешний ключ, вы можете использовать метод `dropForeign`, передав в качестве аргумента имя ограничения внешнего ключа, которое нужно удалить. Ограничения внешнего ключа используют то же соглашение об именах, что и индексы. Другими словами, имя ограничения внешнего ключа основано на имени таблицы и столбцов в ограничении, за которым следует суффикс `_foreign`:

    $table->dropForeign('posts_user_id_foreign');

В качестве альтернативы вы можете передать массив, содержащий имя столбца, который содержит внешний ключ, методу `dropForeign`. Массив будет преобразован в имя ограничения внешнего ключа с использованием соглашений об именах ограничений Laravel:

    $table->dropForeign(['user_id']);

<a name="toggling-foreign-key-constraints"></a>
#### Переключение ограничений внешнего ключа

Вы можете включить или отключить ограничения внешнего ключа в своих миграциях, используя следующие методы:

    Schema::enableForeignKeyConstraints();

    Schema::disableForeignKeyConstraints();

    Schema::withoutForeignKeyConstraints(function () {
    // Constraints disabled within this closure...
    });

> [!WARNING]  
> SQLite по умолчанию отключает ограничения внешнего ключа. При использовании SQLite убедитесь, что [включили поддержку внешнего ключа](database#configuration) в вашей конфигурации базы данных, прежде чем пытаться создать их в ваших миграциях.

<a name="events"></a>
## События

Для удобства каждая операция миграции отправляет [событие](/docs/{{version}}/events). Все следующие события расширяют базовый класс `Illuminate\Database\Events\MigrationEvent`:

| Класс                                            | Описание                                           |
|--------------------------------------------------|----------------------------------------------------|
| `Illuminate\Database\Events\MigrationsStarted`   | Вот-вот будет выполнен пакет миграций.             |
| `Illuminate\Database\Events\MigrationsEnded`     | Завершено выполнение пакета миграций.              |
| `Illuminate\Database\Events\MigrationStarted`    | Одна миграция вот-вот будет выполнена.             |
| `Illuminate\Database\Events\MigrationEnded`      | Выполнение одной миграции завершено.               |
| `Illuminate\Database\Events\NoPendingMigrations` | Команда миграции не обнаружила ожидающих миграций. |
| `Illuminate\Database\Events\SchemaDumped`        | Завершена выгрузка схемы базы данных.              |
| `Illuminate\Database\Events\SchemaLoaded`        | Загружена существующая выгрузка схемы базы данных. |
