## Trait

- Sometimes you may need a set of results multiple times throughout the request cycle. (Categories, Tags, etc...)
- Sometimes you need to cache what's already cached.
- Uses a simple string key instead of serializing arguments, KISS.

```
<?php
namespace App\Traits;
trait Memoize
{
    protected static $memoized = [];
    /**
     * Memoize Operation Result
     * @param $key
     * @param \Closure $callback
     * @param bool $refresh
     * @return mixed
     */
    public function memoize($key, \Closure $callback, $refresh = false){
        if(!isset(static::$memoized[$key]) || $refresh){
            return static::$memoized[$key] = $callback();
        }
        return static::$memoized[$key];
    }
}
```

## Interface
```
return $this->memoize('my-key', function () {
    return //some result;
});
```