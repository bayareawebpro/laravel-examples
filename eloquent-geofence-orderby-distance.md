# Closest To



## Model Scope Usage

```php
<?php

Location::query()
    ->furthestFrom(33.0163, -82.3836, 50)
    ->take(2)
    ->get();

Location::query()
    ->closestTo(33.0163, -82.3836, 50)
    ->take(2)
    ->get();

Location::query()
    ->geoFence(33.0163, -82.3836, 50)
    ->orderBy('distance', 'asc')
    ->take(2)
    ->get();
```


## Model Scope Trait

```php
<?php namespace App;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

class Location extends Model
{
    /**
     * Closest Location
     * @param Builder $query
     * @param float $latitude
     * @param float $longitude
     * @param int $distance
     */
    public function scopeGeoFence(Builder $query, float $latitude, float $longitude, int $distance = 10): void
    {
        $sphere = 'ST_Distance_Sphere(point(longitude, latitude),point(?, ?)) * .000621371192';
        $query
            ->selectRaw("*, ROUND({$sphere}, 2) AS distance", [
                $longitude, $latitude,
            ])
            ->whereRaw("{$sphere} < ?", [
                $longitude, $latitude, $distance,
            ])
        ;
    }

    /**
     * Furthest Location
     * @param Builder $query
     * @param float $latitude
     * @param float $longitude
     * @param int $distance
     */
    public function scopeClosestTo(Builder $query, float $latitude, float $longitude, int $distance = 10): void
    {
        $query
            ->scopes(['geoFence' => compact('latitude', 'longitude', 'distance')])
            ->orderBy('distance', 'asc')
        ;
    }

    /**
     * Furthest Location
     * @param Builder $query
     * @param float $latitude
     * @param float $longitude
     * @param int $distance
     */
    public function scopeFurthestFrom(Builder $query, float $latitude, float $longitude, int $distance = 10): void
    {
        $query
            ->scopes(['geoFence' => compact('latitude', 'longitude', 'distance')])
            ->orderBy('distance', 'desc')
        ;
    }
}
```