# Closest To

```php
use Illuminate\Database\Eloquent\Builder;

/**
 * Closest Location
 * @param Builder $query
 * @param float $latitude
 * @param float $longitude
 * @param int $distance
 */
public function scopeClosestTo(Builder $query, float $latitude, float $longitude, int $distance = 10): void
{
    $sphere = 'ST_Distance_Sphere(point(longitude, latitude),point(?, ?)) * .000621371192';
    $query
        ->selectRaw("*, ROUND({$sphere}, 2) AS distance", [
            $longitude, $latitude,
        ])
        ->whereRaw("{$sphere} < ?", [
            $longitude, $latitude, $distance,
        ])
        ->orderBy('distance', 'asc')
    ;
}
```