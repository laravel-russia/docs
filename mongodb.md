---
git: 88b6f0d99c9f89ca38266d9fa0c3b4ea3df3c85e
---

# MongoDB


<a name="introduction"></a>
## Введение

[MongoDB](https://www.mongodb.com/resources/products/fundamentals/why-use-mongodb) — одна из самых популярных документно-ориентированных баз данных NoSQL, используемая из-за высокой нагрузки на запись (полезно для аналитики или IoT (Интернета вещей) и высокой доступности (простота настройки наборов реплик с автоматическим переходом на другой ресурс). Он также может легко сегментировать базу данных для горизонтального масштабирования и имеет мощный язык запросов для выполнения агрегирования, текстового поиска или геопространственных запросов.

Вместо хранения данных в таблицах строк или столбцов, как в базах данных SQL, каждая запись в базе данных MongoDB представляет собой документ, описанный в BSON, двоичном представлении данных. Затем приложения могут получить эту информацию в формате JSON. Он поддерживает широкий спектр типов данных, включая документы, массивы, встроенные документы и двоичные данные.

Прежде чем использовать MongoDB с Laravel, мы рекомендуем установить и использовать пакет `mongodb/laravel-mongodb` через Composer. Пакет `laravel-mongodb` официально поддерживается MongoDB, и хотя MongoDB изначально поддерживается PHP через драйвер MongoDB, пакет [Laravel MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/) обеспечивает более широкую интеграцию с Eloquent и другими функциями Laravel:

```shell
composer require mongodb/laravel-mongodb
```

<a name="installation"></a>
## Установка

<a name="mongodb-driver"></a>
### Драйвер MongoDB

Для подключения к базе данных MongoDB требуется PHP-расширение `mongodb`. Если вы разрабатываете локально, используя [Laravel Herd](https://herd.laravel.com) или устанавливаете PHP через `php.new`, это расширение уже установлено в вашей системе. Однако, если вам нужно установить расширение вручную, вы можете сделать это через PECL:

```shell
pecl install mongodb
```

Для получения дополнительной информации об установке расширения PHP MongoDB ознакомьтесь с [инструкциями по установке расширения PHP MongoDB](https://www.php.net/manual/en/mongodb.installation.php).

<a name="starting-a-mongodb-server"></a>
### Запуск сервера MongoDB

Сервер MongoDB Community можно использовать для локального запуска MongoDB, и он доступен для установки в Windows, macOS, Linux или в виде контейнера Docker. Чтобы узнать, как установить MongoDB, обратитесь к [официальному руководству по установке сообщества MongoDB](https://docs.mongodb.com/manual/administration/install-community/).

Строку подключения к серверу MongoDB можно установить в файле `.env`:

```ini
MONGODB_URI="mongodb://localhost:27017"
MONGODB_DATABASE="laravel_app"
```

Для размещения MongoDB в облаке рассмотрите возможность использования [MongoDB Atlas](https://www.mongodb.com/cloud/atlas).
Чтобы получить доступ к кластеру MongoDB Atlas локально из вашего приложения, вам необходимо [добавить свой собственный IP-адрес в настройках сети кластера](https://www.mongodb.com/docs/atlas/security/add-ip-address-to-list/) в список IP-доступа проекта.

Строку подключения для MongoDB Atlas также можно установить в файле `.env`:

```ini
MONGODB_URI="mongodb+srv://<username>:<password>@<cluster>.mongodb.net/<dbname>?retryWrites=true&w=majority"
MONGODB_DATABASE="laravel_app"
```

<a name="install-the-laravel-mongodb-package"></a>
### Установка пакета Laravel MongoDB

Наконец, используйте Composer для установки пакета Laravel MongoDB:

```shell
composer require mongodb/laravel-mongodb
```

> [!NOTE]  
> Эта установка пакета завершится неудачно, если не установлено расширение PHP `mongodb`. Конфигурация PHP может различаться в CLI и веб-сервере, поэтому убедитесь, что расширение включено в обеих конфигурациях.

<a name="configuration"></a>
## Конфигурация

Вы можете настроить соединение MongoDB через файл конфигурации вашего приложения `config/database.php`. В этот файл добавьте соединение `mongodb`, которое использует драйвер `mongodb`:

```php
'connections' => [
    'mongodb' => [
        'driver' => 'mongodb',
        'dsn' => env('MONGODB_URI', 'mongodb://localhost:27017'),
        'database' => env('MONGODB_DATABASE', 'laravel_app'),
    ],
],
```

<a name="features"></a>
## Функции

После завершения настройки вы можете использовать пакет `mongodb` и подключение к базе данных в своем приложении, чтобы использовать множество мощных функций:

- [Используя Eloquent](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/eloquent-models/), модели можно хранить в коллекциях MongoDB. В дополнение к стандартным функциям Eloquent пакет Laravel MongoDB предоставляет дополнительные функции, такие как встроенные связи. Пакет также обеспечивает прямой доступ к драйверу MongoDB, который можно использовать для выполнения таких операций, как необработанные запросы и конвейеры агрегации.
- [Написание сложных запросов](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/query-builder/) с помощью построителя запросов.
- `mongodb` [драйвер кэша](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/cache/) оптимизирован для использования функций MongoDB, таких как индексы TTL, для автоматической очистки кэша с истекшим сроком действия. записи.
- [Отправка и обработка заданий в очереди](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/queues/) с помощью драйвера очереди `mongodb`.
- [Хранение файлов в GridFS](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/gridfs/), через [адаптер GridFS для Flysystem](https://flysystem.thephpleague.com/docs/adapter/gridfs/).
— Большинство сторонних пакетов, использующих подключение к базе данных или Eloquent, можно использовать с MongoDB.

Чтобы продолжить изучение использования MongoDB и Laravel, обратитесь к [краткому руководству по началу работы с MongoDB](https://www.mongodb.com/docs/drivers/php/laravel-mongodb/current/quick-start/).
