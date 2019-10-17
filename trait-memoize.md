## Trait

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