
# Custom Logger Instance

Sometimes you might want to log something specifically, possibly for a customer or client.

```php
<?php
use App\Logs\Logger;

Logger::make('auth-log')->write('info','User Logged In', request()->user()->getLoggerAttributes());
```

```php
<?php declare(strict_types=1);

namespace App\Logs;

use Monolog\Formatter\LineFormatter;
use Monolog\Handler\FirePHPHandler;
use Monolog\Handler\StreamHandler;
use Monolog\Logger as Monolog;
use Illuminate\Support\Str;

class Logger {

    const FORMAT = "[%datetime%] %channel%.%level_name%: %message% %context%\n";

    public function __construct(string $fileName) 
    {
        $handler = new StreamHandler(storage_path('/logs/'.Str::slug($fileName).'.log'), 100);
        $handler->setFormatter(new LineFormatter(static::FORMAT));
        $this->logger = tap(new Monolog($fileName))->pushHandler($handler);
    }

    public static function make(string $fileName): Logger
    {
        if(!app()->bound("logger.$fileName")){
            app()->singleton("logger.$fileName", function() use ($fileName){
                return app(static::class, $fileName);
            });
        }
        return app("logger.$fileName");
    }

	public function write(string $level, string $message, array $data = [])
    {
        $this->logger->$level($message,$data);
	}
}
```