# Dialogs for Telegram Bot SDK

<a href="https://github.com/okaufmann/telegram-bot-dialogs/releases"><img src="https://img.shields.io/github/release/okaufmann/telegram-bot-dialogs.svg?style=for-the-badge" alt="Latest Version"/></a>
<a href="https://packagist.org/packages/okaufmann/telegram-bot-dialogs"><img src="https://img.shields.io/packagist/dt/okaufmann/telegram-bot-dialogs.svg?style=for-the-badge" alt="Total Downloads"/></a>

The extension for Telegram Bot API PHP SDK that allows to implement dialogs in bots

This library allows to make simple dialogs for your Telegram bots that based on the [Telegram Bot SDK](https://github.com/irazasyed/telegram-bot-sdk).

## Installation

You can easy install the package using Composer:

    composer require okaufmann/telegram-bot-dialogs

You can publish the config-file with:

    php artisan vendor:publish --provider="BotDialogs\DialogsServiceProvider" --tag="config"

Each dialog should be implemented as class that extends basic Dialog as you can see in example bellow:

```php
use BotDialogs\Dialog;

class HelloDialog extends Dialog
{
    // Array with methods that contains logic of dialog steps.
    // The order in this array defines the sequence of execution.
    protected $steps = ['hello', 'fine', 'bye'];

    public function hello()
    {
        $this->telegram->sendMessage([
            'chat_id' => $this->getChat()->getId(),
            'text' => 'Hello! How are you?'
        ]);
    }

    public function bye()
    {
        $this->telegram->sendMessage([
            'chat_id' => $this->getChat()->getId(),
            'text' => 'Bye!'
        ]);
        $this->jump('hello');
    }

    public function fine()
    {
        $this->telegram->sendMessage([
            'chat_id' => $this->getChat()->getId(),
            'text' => 'I\'m OK :)'
        ]);
    }
}
```

For initiate new dialog you have to use Dialogs class instance to add new dialog implementation. And for execute the first and next steps you have to call Dialogs::proceed() method with update object as an argument. Also it is possible to use dialogs with Telegram commands and DI through type hinting.

```php
use Telegram\Bot\Commands\Command;
use BotDialogs\Dialogs;
use App\Dialogs\HelloDialog;

class HelloCommand extends Command
{
    protected $name = 'hello';

    protected $description = 'Just say "Hello" and ask few questions';

    public function __construct(Dialogs $dialogs)
    {
        $this->dialogs = $dialogs;
    }

    public function handle($arguments)
    {
        $this->dialogs->add(new HelloDialog($this->update));
    }
}
```

And code in the webhook controller

```php
use Telegram\Bot\Api;
use BotDialogs\Dialogs;

// ...

public function __construct(Api $telegram, Dialogs $dialogs)
{
  $this->telegram = $telegram;
  $this->dialogs = $dialogs;
}

// ...

$update = $this->telegram->commandsHandler(true);

if (!$this->dialogs->exists($update)) {
  // Do something if there are no existing dialogs
} else {
  // Call the next step of the dialog
  $this->dialogs->proceed($update);
}
```

For storing dialog information(also for the data that pushed by the Dialog::remember() method) using Redis.

## Advanced definition of the dialog steps

You can define default text answers for your dialog steps. For this you have to define the step as an array with name and response fields.

```php
class HelloDialog extends Dialog
{
    protected $steps = [
        [
            'name' => 'hello',
            'response' => 'Hello my friend!'
        ],
        'fine',
        'bye'
    ];

    // ...
}
```

In this case, if you don't need any logic inside the step handler - you can don't define it. Just put the response inside the step definition. It works good for welcome messages, messages with tips/advices and so on. If you want format response with markdown, just set `markdown` field to `true`.

Also, you can control dialog direction in step by defining `jump` and `end` fields. `jump` acts as `jump()` method - dialog jumps to particular step. `end` field, is set to `true`, ends dialog after current step.

Also, you can use `is_dich` (is it a dichotomous question) option of the step. If this option set to true, you can use `yes` and `no` fields of the Dialog instance to check user answer. For example:

```php
class HelloDialog extends Dialog
{
    protected $steps = [
        [
            'name' => 'hello',
            'response' => 'Hello my friend! Are you OK?'
        ],
        [
            'name' => 'answer',
            'is_dich' => true
        ],
        'bye'
    ];

    public function answer()
    {
        if ($this->yes) {
            // Send message "I am fine, thank you!"
        } elseif ($this->no) {
            // Send message "No, I am got a sick :("
        }
    }
}
```

In the `config/dialogs.php` you can modify aliases for yes/no meanings.

Often in dichotomous question you only need to send response and jump to another step. In this case, you can define steps with responses and set their names as values of 'yes', 'no' or 'default' keys of dichotomous step. For example:

```php
class HelloDialog extends Dialog
{
    protected $steps = [
        [
            'name' => 'hello',
            'response' => 'Hello my friend! Are you OK?'
        ],
        [
            'name' => 'answer',
            'is_dich' => true,
            'yes' => 'fine',
            'no' => 'sick',
            'default' => 'bye'
        ],
        [
            'name' => 'fine',
            'response' => 'I am fine, thank you!',
            'jump' => 'bye',
        ],
        [
            'name' => 'sick',
            'response' => 'No, I am got a sick :(',
        ],
        'bye'
    ];
}
```

## Access control with in dialogs

You can inherit AuthorizedDialog class and put Telegram usernames into \$allowedUsers property. After that just for users in the list will be allowed to start the dialog.

## Available methods of the _Dialog_ class

- `start()` - Start the dialog from the first step
- `proceed()` - Proceed the dialog to the next step
- `end()` - End dialog
- `jump($step)` - Jump to the particular step
- `remember($value)` - Remember some information for the next step usage (For now just a "short" memory works, just for one step)
- `isEnd()` - Check the end of the dialog

## Available methods of the _Dialogs_ class

- `add(Dialog $dialog)` - Add the new dialog
- `get(Telegram\Bot\Objects\Update $update)` - Returns the dialog object for the existing dialog
- `proceed(Telegram\Bot\Objects\Update $update)` - Run the next step handler for the existing dialog
- `exists(Telegram\Bot\Objects\Update $update)` - Check for existing dialog

## Steps configuration in separate files

You can define dialog configuration in separate yaml or php files. To do this, set `scenarios` in dialogs configuration file, using dialog class name as key and path to config file as value, for example:

```php
'scenarios' => [
        HelloDialog::class => base_path('config/dialogs/hello.yml')
],
```

Configuration from files in production environment stored in default cache instance. Because of this, you shall add `php artisan cache:clear` to your deployment script.
