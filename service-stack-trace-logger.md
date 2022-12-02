### Context Aware Stack Trace Logger

- Limit the stack trace to only application namespaces and paths.  
- Include or exclude namespaces and path segments to get short and relevant traces.

## Usage

```php

 $tracer = StackTrace::make()
    ->exclude('Wip')
    ->include([
    'Tests\Forms', 
    'vendor/laravel', 
    'Illuminate'
]);

$tracer->record(new \Exception('Test'), [
    'my-context' => 'my data'
]);
```

## Service Class

```php
<?php declare(strict_types=1);

namespace App\Services;

use Psr\Log\LoggerInterface;

use Illuminate\Support\Str;
use Illuminate\Support\Arr;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\App;

class StackTrace
{
    protected array $excluded = ['Middleware'];
    protected array $included = ['App', 'app'];

    public static function make(): static
    {
        return App::make(static::class);
    }

    public function __construct(protected LoggerInterface $logger)
    {
    }

    public function record(\Throwable $exception, array $context = []): self
    {
        $this->logger->error(
            message: $exception->getMessage(),
            context: [
                'context' => $context,
                'trace' => static::limitTrace($exception->getTrace())->toArray()
            ]
        );
        return $this;
    }

    protected function limitTrace(array $trace): Collection
    {
        return Collection::make($trace)->filter(function (array $trace) {
            $value = Str::of(Arr::get($trace, 'class') ?? Arr::get($trace, 'file'));
            return $value->contains($this->included) && !$value->contains($this->excluded);
        });
    }

    public function include(string|array $namespacesOrPathSegments = []): self
    {
        $this->included = array_merge($this->included, Arr::wrap($namespacesOrPathSegments));
        $this->excluded = array_diff($this->excluded, $this->included);
        return $this;
    }

    public function exclude(string|array $namespacesOrPathSegments = []): self
    {
        $this->excluded = array_merge($this->excluded, Arr::wrap($namespacesOrPathSegments));
        $this->included = array_diff($this->included, $this->excluded);
        return $this;
    }
}
```
