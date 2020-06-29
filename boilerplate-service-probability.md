# Probability Service

## Weighted Random Strategy

Guess a random result using weight to favor a specific distribution of predicatable outcomes.

```php
$color = RandomWeighted::guess([
  'orange' => 2.5, // ~25% chance
  'green' => 3.5, // ~35% chance
  'red' => 4.0, // ~40% chance
]);
```

### Practical Example:

Ads with highest price will be shown more often:

```php
$slug = RandomWeighted::guess([
  'politics' => 250, 
  'sports' => 240,
  'tech' => 190,
]);

Advertiser::category($slug)->inRandomOrder()->first();
```

### Simulation: Totals over x rounds.
```php
$results = RandomWeighted::simulation([
  "a" => 2, 
  "b" => 3,
  "c" => 1,
], 100);

// Totals over 100 rounds.
[
  "a" => ?, 
  "b" => ?,
  "c" => ?,
]
```

### Service Class
```php
<?php declare(strict_types=1);

namespace App\Services\Probability;

use Illuminate\Support\Collection;

class RendomWeighted
{
    /**
     * Random Weighted Key
     * @param array $weights
     * @return mixed|null
     * @source
     */
    public static function guess(array $weights)
    {
        $sum = 0;
        $index = 0;
        $weights = Collection::make($weights);
        $random = mt_rand(1, $weights->sum());
        $weights->each(function ($weight) use (&$sum, &$index, $random) {
            if (($sum += $weight) >= $random) {
                return false;
            }
            $index++;
        });
        return $weights->keys()->get($index);
    }

    /**
     * Sum Rounds of Random Weights
     * @param array $weights
     * @param int $rounds
     * @return Collection
     */
    public static function simulation(array $weights, int $rounds = 1000)
    {
        $results = Collection::times($rounds, fn() => static::guess($weights));
        return Collection::make($weights)->map(function ($weight, $key) use ($results) {
            return $results->filter(fn($result) => $result === $key)->count();
        });
    }
}
```
