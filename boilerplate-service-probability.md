# Probability Service

Useful for A/B Testing, User Segmenting, Weighted Strategies etc... 

## Random Weighted Prediction

Guess a random result using weight to favor a specific distribution of predicatable outcomes.

```php
$color = RandomWeighted::prediction([
  'orange' => 2.5, // ~25% chance
  'green' => 3.5, // ~35% chance
  'red' => 4.0, // ~40% chance
]);
```

### Practical Example:

Ads with higher prices will be shown more often then ads with lower prices:

```php
$slug = RandomWeighted::prediction([
  'politics' => 250, 
  'sports' => 240,
  'tech' => 190,
]);

Advertisement::category($slug)->inRandomOrder()->first();
```

### Simulation: Totals over x rounds.
```php
$results = RandomWeighted::simulation(100, [
  "a" => 2, 
  "b" => 3,
  "c" => 1,
]);

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

class RandomWeighted
{
    /**
     * Random Weighted Key
     * @param array $weights
     * @return string|null
     * @source
     */
    public static function prediction(array $weights): ?string
    {
        $sum = 0;
        $weights = Collection::make($weights)->sort();
        $random = mt_rand(1, $weights->sum());
        foreach($weights as $key => $weight){
             if (($sum += $weight) >= $random) {
                return $key;
            }
        }
    }

    /**
     * Sum Rounds of Random Weights
     * @param array $weights
     * @param int $rounds
     * @return Collection
     */
    public static function simulation(int $rounds, array $weights): Collection
    {
        $results = Collection::times($rounds, function() use ($weights){ 
            return static::prediction($weights);
        });
        return Collection::make($weights)->map(function ($weight, $key) use ($results) {
            return $results->filter(function($result) use ($key){
                return $result === $key;
            })->count();
        });
    }
}
```
