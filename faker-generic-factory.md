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

$invalidState = ContactSubmission::factory()->invalid()->make();

```

## Factory Class
```php
<?php declare(strict_types=1);

namespace Tests\Factories;

class ContactSubmission extends Factory
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
    
    /**
     * Define an alternate state.
     */
    public function invalid(): Factory
    {
        return $this->state(function(array $attributes){
            unset(
                $attributes['name'],
                $attributes['email'],
                $attributes['message'],
                $attributes['captcha'],
            );
            return $attributes;
        });
    }
}

```

## Abstract Factory Class

```php
<?php declare(strict_types=1);

namespace Tests\Factories;

use Faker\Generator as Faker;
use Illuminate\Support\Arr;
use Illuminate\Support\Facades\App;
use Illuminate\Support\LazyCollection;

abstract class Factory
{
    protected int $count = 0;
    protected array $states = [];

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
     * Queue state callback.
     */
    public function state(\Closure $closure): self
    {
        $this->states[] = $closure;
        return $this;
    }

    /**
     * Make new generator instance.
     */
    public function make(array $attributes = []): array|LazyCollection
    {
        return $this->times > 1 ? $this->generator($attributes) : $this->generate($attributes);
    }

    /**
     * Generate one instance.
     */
    protected function generate(array $attributes): array
    {
        $state = $this->definition();

        foreach ($this->states as $callback) {
            $state = call_user_func($callback, $state);
        }

        return array_merge($state, $attributes);
    }

    /**
     * Generate multiple instances.
     */
    protected function generator(array $attributes): LazyCollection
    {
        return LazyCollection::make(function () use ($attributes) {
            while ($this->count < $this->times) {
                $this->count++;
                yield $this->generate($attributes);
            }
            $this->count = 0;
        });
    }
}

```
