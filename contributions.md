---
git: 38fa9c370f19c1bfad16eb1b9e83884f94cf526a
---

# Рекомендации по участию

<a name="bug-reports"></a>
## Отчеты об ошибках

Чтобы стимулировать активное сотрудничество, Laravel настоятельно рекомендует запросы на слияние, а не только отчеты об ошибках. Запросы на слияние будут рассматриваться только в том случае, если они помечены как «готовые к рассмотрению» (не в состоянии «черновик») и все тесты для новых функций пройдены. Устаревшие неактивные запросы на слияние, оставшиеся в состоянии «черновик», будут закрыты через несколько дней.

Однако, если вы отправляете отчет об ошибке, ваш тикет о проблеме должен содержать заголовок и четкое описание проблемы. Вы также должны включить как можно больше релевантной информации и образец кода, демонстрирующий проблему. Цель отчета об ошибке – упростить для вас и окружающих воспроизведение ошибки и разработать исправления.

Помните, отчеты об ошибках создаются в надежде, что другие с той же проблемой смогут сотрудничать с вами при ее решении. Не ожидайте, что отчет об ошибке автоматически сподвигнет на какие-либо действия или что другие будут прыгать, чтобы исправить ее. Создание отчета об ошибке поможет вам и другим начать работу по устранению проблемы. Если хотите внести свой вклад, то вы можете помочь, исправив [любые ошибки, перечисленные в нашем трекере тикетов ошибок](https://github.com/issues?q=is%3Aopen+is%3Aissue+label%3Abug+user%3Alaravel). Вы должны пройти аутентификацию в GitHub, чтобы просмотреть все проблемы Laravel.

Если вы заметили неправильное использование DocBlock, предупреждения PHPStan или IDE при работе с Laravel, не создавайте issue на GitHub. Вместо этого, сделайте запрос на слияние для исправления проблемы.

В управлении исходным кодом Laravel используется GitHub, и для каждого проекта есть репозитории:

<!-- <div class="content-list" markdown="1"> -->

- [Приложение Laravel](https://github.com/laravel/laravel)
- [Логотипы Laravel](https://github.com/laravel/art)
- [Laravel Breeze](https://github.com/laravel/breeze)
- [Документация Laravel](https://github.com/laravel/docs)
- [Пакет Laravel Dusk](https://github.com/laravel/dusk)
- [Пакет Laravel Cashier Stripe](https://github.com/laravel/cashier)
- [Пакет Laravel Cashier Paddle](https://github.com/laravel/cashier-paddle)
- [Пакет Laravel Echo](https://github.com/laravel/echo)
- [Пакет Laravel Envoy](https://github.com/laravel/envoy)
- [Пакет Laravel Folio](https://github.com/laravel/folio)
- [Фреймворк Laravel](https://github.com/laravel/framework)
- [Пакет Laravel Homestead](https://github.com/laravel/homestead) ([Скрипты для сборки](https://github.com/laravel/settler))
- [Пакет Laravel Horizon](https://github.com/laravel/horizon)
- [Пакет Laravel Jetstream](https://github.com/laravel/jetstream)
- [Пакет Laravel Passport](https://github.com/laravel/passport)
- [Пакет Laravel Pennant](https://github.com/laravel/pennant)
- [Пакет Laravel Pint](https://github.com/laravel/pint)
- [Пакет Laravel Prompts](https://github.com/laravel/prompts)
- [Пакет Laravel Reverb](https://github.com/laravel/reverb)
- [Пакет Laravel Sail](https://github.com/laravel/sail)
- [Пакет Laravel Sanctum](https://github.com/laravel/sanctum)
- [Пакет Laravel Scout](https://github.com/laravel/scout)
- [Пакет Laravel Socialite](https://github.com/laravel/socialite)
- [Пакет Laravel Telescope](https://github.com/laravel/telescope)
- [Исходники официального сайта Laravel](https://github.com/laravel/laravel.com)

<!-- </div> -->

<a name="support-questions"></a>
## Вопросы поддержки

Трекеры с тикетами проблем Laravel на GitHub не предназначены для предоставления помощи или поддержки Laravel. Вместо этого используйте один из следующих каналов:

<!-- <div class="content-list" markdown="1"> -->

- [Обсуждения на GitHub](https://github.com/laravel/framework/discussions)
- [Форум Laracasts](https://laracasts.com/discuss)
- [Форум Laravel.io](https://laravel.io/forum)
- [StackOverflow](https://stackoverflow.com/questions/tagged/laravel)
- [Discord](https://discord.gg/laravel)
- [Larachat](https://larachat.co)
- [IRC](https://web.libera.chat/?nick=artisan&channels=#laravel)

<!-- </div> -->

<a name="core-development-discussion"></a>
## Обсуждение разработки ядра

Вы можете предлагать новый функционал или улучшения к уже существующему поведению в репозитории фреймворка Laravel на [доске обсуждений GitHub](https://github.com/laravel/framework/discussions). Если вы предлагаете новый функционал, то, пожалуйста, будьте готовы реализовать по крайней мере часть кода, который потребуется для его завершения.

Неформальное обсуждение ошибок, нового функционала и реализаций существующего происходит на канале `#internals` сервера [Laravel Discord](https://discord.gg/laravel). Тейлор Отвелл, сопровождающий Laravel, обычно присутствует на канале в будние дни с 8:00 до 17:00 (UTC-06:00 или Америка / Чикаго) и от случая к случаю – в остальное время.

<a name="which-branch"></a>
## Какую ветку выбрать при запросах слияния?

**Все** исправления ошибок должны быть отправлены в последнюю версию, которая поддерживает исправления ошибок (на данный момент `11.x`). Исправления ошибок **никогда** не должны отправляться в ветку `master`, если они не исправляют функции, которые существуют только в предстоящем выпуске.

**Минорный** функционал, **полностью обратно совместимый** с текущим релизом, может быть отправлен в последнюю стабильную ветку (в настоящее время `11.x`)..

**Мажорный** новый функционал или функционал с изменениями, приводящими к нарушению обратной совместимости, должен всегда отправляться в ветку `master`, содержащую предстоящий релиз.

<a name="compiled-assets"></a>
## Скомпилированные ресурсы исходников

Если вы отправляете изменение, которое повлияет на скомпилированные файлы, например, касательно файлов в `resources/css` или` resources/js` репозитория `laravel/laravel`, то не включайте в коммит эти скомпилированные файлы. Из-за большого размера они не могут быть реально рассмотрены сопровождающим. Это может быть использовано как способ внедрения вредоносного кода в Laravel. Чтобы предотвратить это, все скомпилированные файлы будут сгенерированы и включены в коммит сопровождающими Laravel.

<a name="security-vulnerabilities"></a>
## Уязвимости безопасности

Если вы обнаружите уязвимость в системе безопасности Laravel, отправьте электронное письмо Тейлору Отвеллу по адресу <a href="mailto:taylor@laravel.com">taylor@laravel.com</a>. Все уязвимости безопасности будут незамедлительно устранены.

<a name="coding-style"></a>
## Стиль кодирования

Laravel следует стандарту кодирования [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide) и стандарту автозагрузки [PSR- 4](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader).

<a name="phpdoc"></a>
### PHPDoc

Ниже приведен пример валидного блока документации Laravel. Обратите внимание, что за атрибутом `@param` идут два пробела, тип аргумента, еще два пробела и, наконец, имя переменной:

    /**
     * Register a binding with the container.
     *
     * @param  string|array  $abstract
     * @param  \Closure|string|null  $concrete
     * @param  bool  $shared
     * @return void
     *
     * @throws \Exception
     */
    public function bind($abstract, $concrete = null, $shared = false)
    {
        // ...
    }

Когда атрибуты `@param` или `@return` являются избыточными из-за использования нативных типов, их можно удалить:

    /**
     * Execute the job.
     */
    public function handle(AudioProcessor $processor): void
    {
        //
    }

Однако, когда нативный тип является обобщенным, укажите его через использование атрибутов `@param` или `@return`:

    /**
     * Get the attachments for the message.
     *
     * @return array<int, \Illuminate\Mail\Mailables\Attachment>
     */
    public function attachments(): array
    {
        return [
            Attachment::fromStorage('/path/to/file'),
        ];
    }

<a name="styleci"></a>
### StyleCI

Не волнуйтесь, если стиль вашего кода не идеален! [StyleCI](https://styleci.io/) автоматически объединит любые исправления стиля после слияния вашего запроса с репозиторием Laravel. Это позволяет нам сосредоточиться на содержании вашего вклада, а не на стиле кода.

<a name="code-of-conduct"></a>
## Нормы поведения

Кодекс поведения Laravel основан на кодексе поведения Ruby. О любых нарушениях кодекса поведения можно сообщить Тейлору Отвеллу (taylor@laravel.com):

<!-- <div class="content-list" markdown="1"> -->

- Участники будут терпимо относиться к противоположным взглядам.
- Участники должны гарантировать, что их язык и действия не содержат личных нападок и пренебрежительных личных замечаний.
- Толкуя слова и действия других, участники всегда должны исходить из добрых намерений.
- Не допускается поведение, которое можно обоснованно считать преследованием.

<!-- </div> -->
