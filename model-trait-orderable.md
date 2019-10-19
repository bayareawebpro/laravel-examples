# Orderable Trait (WIP, more unit tests coming)

> Contributor: https://github.com/codemonkey76

> Refactoring & Testing: https://github.com/bayareawebpro

## Schema

```
$table->bigInteger('order')->default(0);
$table->string('group')->default(null); //Any type
```

## Model

```
<?php namespace App;

use App\Traits\Orderable;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use Orderable;

    protected $orderGroup = 'group';

    protected $attributes = [
        'group'=> 'my-group'
    ];

    protected $fillable = [
        'order',
        'group',
    ];

    // Required for UnitTesting with SqlLite
    protected $casts = [
        'order'=> 'int'
    ];
}

```

## Trait
```
<?php namespace App\Traits;

use Illuminate\Support\Facades\DB;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

trait Orderable
{
    //protected $orderGroup = null;

    public function incrementOrder(): bool
    {
        if ($this->getAttribute('isHighestOrder')) return false;
        DB::transaction(function () {
            static::newQuery()
                ->scopes(['orderGroup'])
                ->where('order', $this->getAttribute('order') + 1)
                ->whereKeyNot($this->getKey())
                ->decrement('order');
            $this->increment('order');
            $this->save();
        });
        return true;
    }

    public function decrementOrder(): bool
    {
        if ($this->getAttribute('isLowestOrder')) return false;
        DB::transaction(function () {
            static::newQuery()
                ->scopes(['orderGroup'])
                ->where('order', $this->getAttribute('order') - 1)
                ->whereKeyNot($this->getKey())
                ->increment('order');
            $this->decrement('order');
            $this->save();
        });
        return true;
    }

    public function updateOrderGroup(?string $orderGroup = null): bool
    {
        if(isset($orderGroup)){
            DB::transaction(function () use ($orderGroup){
                static::newQuery()
                    ->scopes(['orderGroup', 'higherThan'])
                    ->decrement('order');
                $this->setAttribute($this->orderGroup, $orderGroup);
                $this->setAttribute('order', null);
                $this->save();
            });
            return true;
        }
        return false;
    }
    
    public function scopeOrderGroup(Builder $builder, ?string $orderGroup = null): void
    {
        $orderGroup = $orderGroup ?? $this->orderGroup;
        if (isset($orderGroup)) {
            $builder->where($orderGroup, $this->getAttribute($orderGroup));
        }
    }

    public function scopeHigherThan(Builder $builder, ?int $order = null): void
    {
        $builder
            ->scopes(['orderGroup'])
            ->where('order', '>', $order ?? $this->getAttribute('order'))
            ->orderBy('order', 'ASC');
    }
    
    public function scopeLowerThan(Builder $builder, ?int $order = null): void
    {
        $builder
            ->scopes(['orderGroup'])
            ->where('order', '<',$order ?? $this->getAttribute('order'))
            ->orderBy('order', 'DESC');
    }

    public function getHighestOrderAttribute(): int
    {
        return (int) static::newQuery()->scopes(['orderGroup'])->max('order');
    }

    public function getLowestOrderAttribute(): int
    {
        return (int) static::newQuery()->scopes(['orderGroup'])->min('order');
    }

    public function getIsLowestOrderAttribute(): bool
    {
        return (bool)($this->getAttribute('lowestOrder') === $this->getAttribute('order'));
    }

    public function getIsHighestOrderAttribute(): bool
    {
        return (bool)($this->getAttribute('highestOrder') === $this->getAttribute('order'));
    }
    
    public static function bootOrderable()
    {
        static::saving(function (Model $model) {
            if(empty($model->getAttribute('order'))){
                $model->setAttribute('order', $model->getAttribute('highestOrder') + 1);
            }
        });
    }
}
```

## Unit Test
```
<?php
namespace Tests\Unit;

use App\Post;
use Illuminate\Support\Collection;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class OrderableTest extends TestCase
{
    use RefreshDatabase;

    /**
     * A basic test example.
     * @return void
     */
    public function testBasicTest()
    {
        Post::query()->delete();
        /**
         * @var $postA Post
         * @var $postB Post
         * @var $postC Post
         * @var $postD Post
         */
        $postA = tap(Post::create(['group' => 'base']))->save();
        $postB = tap(Post::create(['group' => 'base']))->save();
        $postC = tap(Post::create(['group' => 'base']))->save();
        $postD = tap(Post::create(['group' => 'base']))->save();
        $postE = tap(Post::create(['group' => 'alternate']))->save();

        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();


        $this->assertTrue($postA->lowestOrder === 1, 'A => 1 lowestOrder');
        $this->assertTrue($postA->highestOrder === 4, 'A => 4  highestOrder');
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'C 3');
        $this->assertTrue($postA->isLowestOrder, 'A self lowest');
        $this->assertTrue($postD->isHighestOrder, 'D self highest');

        $this->assertTrue($postE->order === 1, 'E 1');
        $this->assertTrue($postE->group === 'alternate', 'E group');
        $this->assertTrue($postE->isLowestOrder, 'E self lowest');
        $this->assertTrue($postE->isHighestOrder, 'E self highest');


        $postE->updateOrderGroup('base');
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();


        $this->assertTrue($postE->order === 5, 'E 5');
        $this->assertTrue($postE->group === 'base', 'E group');
        $this->assertFalse($postE->isLowestOrder, 'E self lowest');
        $this->assertTrue($postE->isHighestOrder, 'E self highest');

        //Ignore Decrement of Lowest
        $postA->decrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');
        $this->assertTrue($postE->order === 5, 'E 5');

        //Increment Lowest Value
        $postA->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $this->assertTrue($postA->order === 2, 'A 2');
        $this->assertTrue($postB->order === 1, 'B 1');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');
        $this->assertTrue($postE->order === 5, 'E 5');

        //Increment New Lowest Value
        $postB->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');
        $this->assertTrue($postE->order === 5, 'E 5');

        //Increment 2nd Highest Value
        $postC->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 4, 'C 4');
        $this->assertTrue($postD->order === 3, 'D 3');
        $this->assertTrue($postE->order === 5, 'E 5');

        //Increment 2nd Highest Value
        $postD->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');
        $this->assertTrue($postE->order === 5, 'E 5');


        dump(Collection::make([
            "A" => $postA->order,
            "B" => $postB->order,
            "C" => $postC->order,
            "D" => $postD->order,
            "E" => $postE->order,
        ])->sort());
        //Ignore Increment of Highest
        $postE->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');
        $this->assertTrue($postE->order === 5, 'E 5');

        dump(Collection::make([
            "A" => $postA->order,
            "B" => $postB->order,
            "C" => $postC->order,
            "D" => $postD->order,
            "E" => $postE->order,
        ])->sort());
    }
}
```
