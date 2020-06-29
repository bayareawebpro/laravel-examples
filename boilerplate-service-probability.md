# Probability Service

```php
$pick = Probability::randomWeighted([
  "a" => 1, 
  "b" => 3,
  "c" => 2,
]);
```

### Random Weighted Key

```php

$contracts = [
  "a" => 1, 
  "b" => 3,
  "c" => 2,
];

Probability::rounds($contracts, 100);

//hits per 100 rounds.
[
  "a" => 23, 
  "b" => 61,
  "c" => 16,
]
```

### Service Class
```php
<?php

namespace App\Leads;

use Illuminate\Support\Collection;

class Probability
{
    /**
     * Random Key
     * @param array $items
     * @return mixed|null
     */
    public static function randomKey(array $items)
    {
        return Collection::make($items)->keys()->random();
    }

    /**
     * Random Value
     * @param array $items
     * @return mixed|null
     */
    public static function randomValue(array $items)
    {
        return Collection::make($items)->values()->random();
    }

    /**
     * Random Element
     * @param array $items
     * @return mixed|null
     */
    public static function randomElement(array $items)
    {
        $randomKey = static::randomKey($items);
        return Collection::make($items)->reject(fn($item, $key) => $key !== $randomKey)->all();
    }

    /**
     * Random Weighted Element
     * @param array $weights
     * @return mixed|null
     * @source
     */
    public static function randomWeighted(array $weights)
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
    public static function rounds(array $weights, int $rounds = 1000)
    {
        $results = Collection::make();
        foreach (range(1, $rounds) as $index) {
            $results->push(Probability::randomWeighted($weights));
        }
        return Collection::make($weights)->map(function ($value, $key) use ($results) {
            return $results->filter(fn($value) => $value === $key)->count();
        });
    }
}
```
