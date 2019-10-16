```
$service = Service::make(config('my.config'));

$service->name //configured, computed & cached;

$service->toArray();
```

```

app()->bind(Service::class, function(){
    return Service::make(config('my.config'));
});

$service = app(Service::class);
```


```
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Str;
use Illuminate\Cache\Repository as Cache;
use Illuminate\Contracts\Support\Arrayable;

class Service implements Arrayable{

    public $id;
    public $key;
    public $name;
    protected $cache;
    
    /**
     * Service constructor.
     * @param Cache $cache
     * @param string $id
     * @param string $key
     * @param string $name
     */
    public function __construct(
        Cache $cache,
        int $id,
        string $key,
        string $name,
    ){
        $this->id = $id;
        $this->key = $key;
        $this->name = $name;
        $this->cache = $cache;
    }

    /**
     * Make new instance of self (expects config array entry).
     * @param array $attributes
     * @return \Illuminate\Contracts\Foundation\Application|mixed
     */
    public static function make(array $attributes): Contract
    {
        return app(self::class, $attributes);
    }

    /**
     * Get property value from getter.
     * @param $property
     * @return mixed|null
     */
    public function __get($property)
    {
        $getter = "get" . Str::studly($property);
        if (method_exists($this, $getter)) {
            return $this->$getter();
        }
        return $this->$property ?? null;
    }

    /**
     * Get a cache key for the contract.
     * @param string $key
     * @return string
     */
    public function cacheKey(string $key): string
    {
        return "{$this->id}:{$key}";
    }

    /**
     * Get a computed property (name).
     * @param string $key
     * @return string
     */
    public function getName(): string
    {
        return $this->cache->forever($this->cacheKey('name'), function(){
            return "{$this->name}::my-special-computed-key";
        });
    }

    /**
     * Get the array representation of the data.
     * @return array
     */
    public function toArray()
    {
        return [
            'id'    => $this->id,
            'name'  => $this->name,
            'key'   => $this->key,
            ...
        ];
    }
```