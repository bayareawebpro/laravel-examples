# Custom Logger Instance

Sometimes you might want to log something specifically, possibly for a customer or client.

Using the Forwards Calls Trait (found in laravel) will allow you to wrap another class, 

We can declare the methods supported by using DocBlocks:

```
@method static Logger debug(string $message, array $data = [])
```

This can be a little confusing to read, but in plain english: 
- exposes static method 
- that returns instance of Logger 
- named debug 
- that accepts a string "message"
- and an array "data"

## Usage
```php
<?php
use App\Services\Logger;

Logger::make('pages-show')->error('Test', request()->all());
```

Output

```
[2019-10-24 07:44:17] pages-show.ERROR: Test {"keywords":"exciting","sort":"asc"}
```

## Implementation

This class uses a "lazy-singleton" so that only one instance of each log file is opened during a request.

```php
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Traits\ForwardsCalls;
use Monolog\Formatter\LineFormatter;
use Monolog\Handler\StreamHandler;
use Monolog\Logger as Monolog;
use Illuminate\Support\Str;

/**
 * @method static Logger debug(string $message, array $data = [])
 * @method static Logger info(string $message, array $data = [])
 * @method static Logger notice(string $message, array $data = [])
 * @method static Logger warning(string $message, array $data = [])
 * @method static Logger error(string $message, array $data = [])
 * @method static Logger critical(string $message, array $data = [])
 * @method static Logger alert(string $message, array $data = [])
 * @method static Logger emergency(string $message, array $data = [])
 */
class Logger
{
    use ForwardsCalls;

    /**
     * @var Monolog
     */
    protected $logger;

    const FORMAT = "[%datetime%] %channel%.%level_name%: %message% %context%\n";

    public function __construct(string $fileName)
    {
        $this->logger = $this->getMonologInstance($fileName);
    }

    /**
     * Close the log handlers.
     */
    public function __destruct()
    {
        $this->logger->close();
    }

    /**
     * Forward calls to Monolog.
     * @param $name
     * @param $arguments
     * @return $this
     */
    public function __call($name, $arguments): self
    {
        $this->forwardCallTo($this->logger, Str::upper($name), $arguments);
        return $this;
    }

    /**
     * Make Instance of Self
     * @param string $fileName
     * @return Logger
     */
    public static function make(string $fileName): Logger
    {
        if(!app()->bound("logger.$fileName")){
            app()->singleton("logger.$fileName", function() use ($fileName){
                return app(static::class, compact('fileName'));
            });
        }
        return app("logger.$fileName");
    }

    /**
     * Get Monolog Instance
     * @param string $fileName
     * @return Monolog
     * @throws \Exception
     */
    protected function getMonologInstance(string $fileName): Monolog
    {
        $handler = new StreamHandler(storage_path('logs/' . Str::slug($fileName) . '.log'));
        $handler->setFormatter(new LineFormatter(static::FORMAT));
        return tap(new Monolog($fileName))->pushHandler($handler);
    }
}
```