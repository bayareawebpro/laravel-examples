# Abstract Faker Factory for Non-Models

## Usage

```php
$array = ContactSubmission::factory()->make([
  'name' => 'other'
]);

$lazyCollection = ContactSubmission::factory(100)->make([
  'name' => 'other'
]);

$itemsArray = $lazyCollection->all();

```

## Factory Class
```php
<?php declare(strict_types=1);

namespace Tests\Factories;

class ContactSubmission extends AbstractFactory
{
    /**
     * Define the default state.
     */
    protected function definition(): array
    {
        return [
            'name' => $this->faker->name(),
            'email' => $this->faker->safeEmail(),
            'message' => $this->faker->sentences(4, true),
        ];
    }
}

```

## Abstract Factory Class

```php
<?php declare(strict_types=1);

namespace Tests\Factories;

use Faker\Generator as Faker;
use Illuminate\Support\Facades\App;
use Illuminate\Support\LazyCollection;

abstract class AbstractFactory
{
    protected int $count = 0;
    protected array $attributes = [];

    public function __construct(protected Faker $faker, protected int $times = 1)
    {
        //
    }

    /**
     * Create a new factory instance.
     */
    public static function factory(int $times = 1): self
    {
        return App::make(static::class, compact('times'));
    }

    /**
     * Define the default state.
     */
    protected function definition(): array
    {
        return [];
    }

    /**
     * Make new instance(s).
     */
    public function make(array $attributes = []): array|LazyCollection
    {
        $this->attributes = $attributes;
        return $this->times > 1 ? $this->generator() : $this->generate();
    }

    /**
     * Generate one instance.
     */
    protected function generate(): array
    {
        return array_merge($this->definition(), $this->attributes);
    }

    /**
     * Generate multiple instances.
     */
    protected function generator(): LazyCollection
    {
        return LazyCollection::make(function (){
            while ($this->count < $this->times) {
                $this->count++;
                yield $this->generate();
            }
            $this->count = 0;
        });
    }
}

```
