# Lazy Generator
- Sometimes you need some fake data on demand, maybe a lot of it...

## Usage
```
use App\Services\LazyGenerator;

$lazyCollection = LazyGenerator::make(100, function(\Faker\Generator $faker){
    return [
        'uuid' => (string) $faker->uuid,
        'name' => (string) $faker->name,
        'email' => (string) $faker->email,
    ];
});
```

## Service Class
```
<?php declare(strict_types=1);

namespace App\Services;

use Faker\Generator;
use Illuminate\Support\LazyCollection;

class LazyGenerator
{
    /**
     * @param int $total
     * @param \Closure $callback
     * @return LazyCollection
     */
    public static function make(int $total, \Closure $callback): LazyCollection
    {
        $count = 0;
        $faker = app(Generator::class);
        return LazyCollection::make(function () use (&$count, $total, $faker, $callback) {
            while ($count < $total) {
                $count++;
                yield $callback($faker);
            }
        });
    }
}
```