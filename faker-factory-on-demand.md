## Define Faker Eloquent Factory on Demand

- Useful during package development.

```
<?php 
namespace App\Tests\Unit;

use Faker\Factory;
use Faker\Generator;
use Illuminate\Database\Eloquent\Factory as EloquentFactory;

class DefaultTest extends TestCase
{
    public function setUp(): void
    {
        parent::setUp();
        $this->app->resolving(EloquentFactory::class, function(EloquentFactory $factory){
            $factory->define(User::class, function (Generator $faker) {
                return [
                    'name' => $faker->name,
                    'email' => $faker->unique()->safeEmail,
                    'email_verified_at' => now(),
                    'password' => $faker->randomAscii,
                    'remember_token' => Str::random(10),
                ];
            });
        });
    }

    private function getTestData()
    {
        return factory(User::class, 100)->make();
    }
}
```