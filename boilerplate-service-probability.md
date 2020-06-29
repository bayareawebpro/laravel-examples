# Probability Service

Build a new random class using weight factoring the randomness.

```php
$pick = new app(Probability::randomWeighted([
  ObjectA::class => 1.5, 
  ObjectB::class => 2.7,
  ObjectC::class => 1.9,
]));
```

### Random Weighted Key

```php

Probability::rounds([
  "a" => 1, 
  "b" => 3,
  "c" => 2,
], 100);

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
