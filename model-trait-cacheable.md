## Example
```
/**
 * Get Reactions Count
 * @return mixed
 */
public function getReactionCountAttribute()
{
    return Cache::rememberForever($this->cacheKey('reactionCount'), function () {
        return (
            $this->reactions()->where('type', 'positive')->count() -
            $this->reactions()->where('type', 'negative')->count()
        );
    });
}

/**
 * Clear the cached attributes.
 * @return void
 */
public function clearCache(): void
{
    Cache::forget($this->cacheKey('reactionCount'));
}

```

## Cacheable Trait
```
<?php declare(strict_types=1);

namespace App\Traits;
use Illuminate\Support\Arr;
use Illuminate\Support\Collection;
use Illuminate\Database\Eloquent\Model;

trait Cacheable{

    /**
     * @param array|string $keys
     * @return string
     */
    public function cacheKey(...$keys): string
    {
        return Collection::make(Arr::wrap($keys))
        ->concat([$this->getKey()])
        ->flatten(1)
        ->join('-');
    }

    /**
     * Boot the Trait
     * @return void
     */
    public static function bootCacheable(){
        static::saving(function(Model $model){
            if(method_exists($model, 'clearCache')){
                $model->clearCache();
            }
        });
    }
}

```

