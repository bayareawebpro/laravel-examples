```
use Faker\Generator;
use Faker\Factory as FakerFactory;

class MyRandomDigit extends \Faker\Provider\Base
{
    public function myRandomDigit()
    {
        return static::randomDigit();
    }
}

$this->app->resolving(\Faker\Generator::class, function (Generator $generator) {
    $generator->addProvider(new MyRandomDigit($generator));
    return $generator;
});
```

```

$faker->myRandomDigit
```
