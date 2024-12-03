---
git: 9f36b02f2c2968ad2c6945df79d9eaf31dfdd224
---

# База данных · Наполнение фиктивными данными

<a name="introduction"></a>
## Введение

Laravel предлагает возможность наполнения вашей базы тестовыми данными с использованием классов-наполнителей. Все классы наполнителей хранятся в каталоге `database/seeders`. Класс `DatabaseSeeder` уже определен по умолчанию. В этом классе вы можете использовать метод `call` для запуска других наполнителей, что позволит вам контролировать порядок наполнения БД.

> [!NOTE]
> При наполнении базы данных автоматически отключается защита [массового присвоения](/docs/{{version}}/eloquent#mass-assignment).

<a name="writing-seeders"></a>
## Написание наполнителей

Чтобы сгенерировать новый наполнитель, используйте команду `make:seeder` [Artisan](artisan). Эта команда поместит новый класс наполнителя в каталог `database/seeders` вашего приложения:

```shell
php artisan make:seeder UserSeeder
```

Класс наполнителя (сидера) по умолчанию содержит только один метод: `run`. Этот метод вызывается при выполнении [команды Artisan](/docs/{{version}}/artisan) `db:seed`. Внутри метода `run` вы можете вставлять данные в свою базу данных так, как вам удобно. Вы можете использовать [строитель запросов (query builder)](/docs/{{version}}/queries) для ручной вставки данных или [фабрики моделей Eloquent](/docs/{{version}}/eloquent-factories).

В качестве примера давайте изменим класс `DatabaseSeeder`, созданный по умолчанию, и добавим выражение вставки фасада `DB` в методе `run`:

    <?php

    namespace Database\Seeders;

    use Illuminate\Database\Seeder;
    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Str;

    class DatabaseSeeder extends Seeder
    {
        /**
         * Запустить наполнение базы данных.
         */
        public function run(): void
        {
            DB::table('users')->insert([
                'name' => Str::random(10),
                'email' => Str::random(10).'@example.com',
                'password' => Hash::make('password'),
            ]);
        }
    }

> [!NOTE]
> В методе `run` вы можете объявить любые необходимые типы зависимостей. Они будут автоматически извлечены и внедрены через [контейнер служб](/docs/{{version}}/container) Laravel.

<a name="using-model-factories"></a>
### Использование фабрик моделей

Конечно, ручное указание атрибутов для каждой модели наполнителя обременительно. Вместо этого вы можете использовать [фабрики моделей](/docs/{{version}}/eloquent-factories) для удобного создания большого количества записей в БД. Сначала просмотрите [документацию фабрики моделей](/docs/{{version}}/eloquent-factories), чтобы узнать, как определить свои фабрики.

Например, давайте создадим 50 пользователей, у каждого из которых будет по одному посту:

    use App\Models\User;

    /**
     * Запустить наполнение базы данных.
     */
    public function run(): void
    {
        User::factory()
                ->count(50)
                ->hasPosts(1)
                ->create();
    }

<a name="calling-additional-seeders"></a>
### Вызов дополнительных наполнителей

Внутри класса `DatabaseSeeder` вы можете использовать метод `call` для запуска других наполнителей. Использование метода `call` позволяет вам разбить ваши наполнители БД на несколько файлов, так что ни один класс наполнителя не станет слишком большим. Метод `call` принимает массив классов, которые должны быть выполнены:

    /**
     * Запустить наполнение базы данных.
     */
    public function run(): void
    {
        $this->call([
            UserSeeder::class,
            PostSeeder::class,
            CommentSeeder::class,
        ]);
    }

<a name="muting-model-events"></a>
### Отключение событий модели

При выполнении сидов (seeds) вы можете захотеть предотвратить моделям отправку событий. Для этого вы можете использовать трейт `WithoutModelEvents`. При его использовании, трейт `WithoutModelEvents` гарантирует, что события модели не будут отправлены, даже если дополнительные сид-классы выполняются с помощью метода `call`:

    <?php

    namespace Database\Seeders;

    use Illuminate\Database\Seeder;
    use Illuminate\Database\Console\Seeds\WithoutModelEvents;

    class DatabaseSeeder extends Seeder
    {
        use WithoutModelEvents;

        /**
         * Запуск сидеров базы данных.
         */
        public function run(): void
        {
            $this->call([
                UserSeeder::class,
            ]);
        }
    }

Этот трейт поможет вам отключить отправку событий модели во время выполнения сидов (seeds).

<a name="running-seeders"></a>
## Запуск наполнителей

Вы можете выполнить команду `db:seed` Artisan для наполнения вашей базы данных. По умолчанию команда `db:seed` запускает класс `Database\Seeders\DatabaseSeeder`, который, в свою очередь, может вызывать другие классы. Однако вы можете использовать параметр `--class`, чтобы указать конкретный класс наполнителя для его индивидуального запуска:

```shell
php artisan db:seed

php artisan db:seed --class=UserSeeder
```

Вы также можете заполнить свою базу данных, используя команду `migrate:fresh` в сочетании с опцией `--seed`, которая удалит все таблицы и перезапустит все миграции. Эта команда полезна для полной перестройки вашей базы данных. Опцию `--seeder` можно использовать для указания конкретного сида (seeder) для выполнения:

```shell
php artisan migrate:fresh --seed

php artisan migrate:fresh --seed --seeder=UserSeeder
```

<a name="forcing-seeding-production"></a>
#### Принудительное наполнение при эксплуатации приложения

Некоторые операции наполнения могут привести к изменению или потере данных. В окружении `production`, чтобы защитить вас от запуска команд наполнения эксплуатируемой базы данных, вам будет предложено подтвердить их запуск. Чтобы заставить наполнители запускаться без подтверждений, используйте флаг `--force`:

```shell
php artisan db:seed --force
```
