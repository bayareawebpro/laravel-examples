## Define a Custom Faker Factory

- Useful for custom data types.

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
```

## FakerServiceProvider
```
$this->app->resolving(\Faker\Generator::class, function (Generator $generator) {
    $generator->addProvider(new MyRandomDigit($generator));
    return $generator;
});
```

## Usage
```
use Faker\Generator as Faker;
$factory->define(Model::class, function (Faker $faker) {
    return [
        'my_field' => $faker->myRandomDigit,
    ];
});
```
