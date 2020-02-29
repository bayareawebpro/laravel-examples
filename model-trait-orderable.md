# Orderable Trait

> Contributor: https://github.com/codemonkey76

> Refactoring & Testing: https://github.com/bayareawebpro

## Schema

```php
$table->bigInteger('order')->default(0);
$table->string('group')->default(null); //Any type
```

## Model

```php
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
```php
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
        DB::beginTransaction();
        static::newQuery()
            ->scopes(['orderGroup'])
            ->where('order', $this->getAttribute('nextOrder'))
            ->decrement('order');
        $this->increment('order');
        $this->save();
        DB::commit();
        return true;
    }

    public function decrementOrder(): bool
    {
        if ($this->getAttribute('isLowestOrder')) return false;
        DB::beginTransaction();
        static::newQuery()
            ->scopes(['orderGroup'])
            ->where('order', $this->getAttribute('previousOrder'))
            ->increment('order');
        $this->decrement('order');
        $this->save();
        DB::commit();
        return true;
    }

    public function updateOrderGroup(?string $orderGroup = null): bool
    {
        if (isset($orderGroup)) {
            DB::beginTransaction();
            static::newQuery()
                ->scopes(['orderGroup', 'higherThan'])
                ->decrement('order');
            $this->setAttribute($this->orderGroup, $orderGroup);
            $this->setAttribute('order', null);
            $this->save();
            DB::commit();
            return true;
        }
        return false;
    }

    public function scopeOrderGroup(Builder $builder, ?string $orderGroup = null): void
    {
        $orderGroup = $orderGroup ?? $this->getAttribute($this->orderGroup);
        if (isset($orderGroup)) {
            $builder->where($this->orderGroup, $orderGroup);
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
            ->where('order', '<', $order ?? $this->getAttribute('order'))
            ->orderBy('order', 'DESC');
    }

    public function getIsLowestOrderAttribute(): bool
    {
        return (bool)($this->getAttribute('lowestOrder') === $this->getAttribute('order'));
    }

    public function getIsHighestOrderAttribute(): bool
    {
        return (bool)($this->getAttribute('highestOrder') === $this->getAttribute('order'));
    }

    public function getLowestOrderAttribute(): int
    {
        return (int)static::newQuery()->scopes(['orderGroup'])->min('order');
    }

    public function getHighestOrderAttribute(): int
    {
        return (int)static::newQuery()->scopes(['orderGroup'])->max('order');
    }

    public function getPreviousOrderAttribute(): int
    {
        return (int)($this->getAttribute('order') - 1);
    }

    public function getNextOrderAttribute(): int
    {
        return (int)($this->getAttribute('order') + 1);
    }

    public static function bootOrderable(): void
    {
        static::saving(function (Model $model) {
            if (empty($model->getAttribute('order'))) {
                $model->setAttribute('order', $model->getAttribute('highestOrder') + 1);
            }
        });
    }
}
```

## Unit Test
```php
<?php
namespace Tests\Unit;

use App\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class OrderableTest extends TestCase
{
    use RefreshDatabase;

    public function test_orders_self_when_created(){
        /**
         * @var $postA Post
         * @var $postB Post
         * @var $postC Post
         * @var $postD Post
         * @var $postE Post
         */
        $postA = Post::create(['group' => 'base']);
        $postB = Post::create(['group' => 'base']);
        $postC = Post::create(['group' => 'base']);
        $postD = Post::create(['group' => 'base']);
        $postE = Post::create(['group' => 'alternate']);
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
    }

    public function test_will_ignore_decrement_when_self_lowest(){
        /**
         * @var $postA Post
         * @var $postB Post
         */
        $postA = Post::create(['group' => 'base', 'order' => 1]);
        $postB = Post::create(['group' => 'base', 'order' => 2]);

        $postA->decrementOrder();
        $postA->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
    }

    public function test_will_ignore_increment_when_self_highest(){
        /**
         * @var $postA Post
         */
        Post::create(['group' => 'base', 'order' => 1]);
        $postB = Post::create(['group' => 'base', 'order' => 2]);

        $postB->incrementOrder();
        $postB->refresh();
        $this->assertTrue($postB->order === 2, 'A 1');
    }

    public function test_will_decrement_next_sibling_if_self_incremented(){
        /**
         * @var $postA Post
         * @var $postB Post
         * @var $postC Post
         */
        $postA = Post::create(['group' => 'base', 'order' => 1]);
        $postB = Post::create(['group' => 'base', 'order' => 2]);
        $postC = Post::create(['group' => 'base', 'order' => 3]);

        $postA->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $this->assertTrue($postA->order === 2, 'A 2');
        $this->assertTrue($postB->order === 1, 'B 1');
        $this->assertTrue($postC->order === 3, 'C 3');
    }

    public function test_will_increment_previous_sibling_if_self_decremented(){
        /**
         * @var $postA Post
         * @var $postB Post
         * @var $postC Post
         */
        $postA = Post::create(['group' => 'base', 'order' => 1]);
        $postB = Post::create(['group' => 'base', 'order' => 2]);
        $postC = Post::create(['group' => 'base', 'order' => 3]);

        $postB->decrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $this->assertTrue($postA->order === 2, 'A 2');
        $this->assertTrue($postB->order === 1, 'B 1');
        $this->assertTrue($postC->order === 3, 'C 3');
    }

    public function test_will_scope_order_group(){
        /**
         * @var $post Post
         */
        Post::query()->delete();
        Post::create(['group' => 'base', 'order' => 1]);
        Post::create(['group' => 'base', 'order' => 2]);
        Post::create(['group' => 'base', 'order' => 3]);
        Post::create(['group' => 'alternate', 'order' => 1]);
        Post::create(['group' => 'alternate', 'order' => 2]);
        Post::create(['group' => 'alternate', 'order' => 3]);
        $post = Post::create(['group' => 'alternate', 'order' => 4]);

        $this->assertSame(7, Post::query()->count(), '7 total in static unscoped query');
        $this->assertSame(4, $post->orderGroup()->count(), '4 total in instance query');
        $this->assertSame(4, Post::query()->orderGroup('alternate')->count(), '4 in static query with param');
        $this->assertSame(3, Post::query()->orderGroup()->count(), '3 in static default query');
    }

    public function test_will_decrement_siblings_and_reorder_self_if_group_changed(){
        /**
         * @var $postD Post
         * @var $postE Post
         * @var $postF Post
         */
        Post::create(['group' => 'base', 'order' => 1]);
        Post::create(['group' => 'base', 'order' => 2]);
        Post::create(['group' => 'base', 'order' => 3]);
        $postD = Post::create(['group' => 'alternate', 'order' => 1]);
        $postE = Post::create(['group' => 'alternate', 'order' => 2]);
        $postF = Post::create(['group' => 'alternate', 'order' => 3]);

        $postE->updateOrderGroup('base');
        $postD->refresh();
        $postE->refresh();
        $postF->refresh();
        $this->assertTrue($postE->order === 4, 'E 4');
        $this->assertTrue($postE->group === 'base', 'E group');
        $this->assertTrue($postE->isHighestOrder, 'E self highest');
        $this->assertFalse($postE->isLowestOrder, 'E self lowest');
        $this->assertTrue($postD->order === 1, 'D 1');
        $this->assertTrue($postF->order === 2, 'F 3');

        $postD->updateOrderGroup('base');
        $postD->refresh();
        $postE->refresh();
        $postF->refresh();
        $this->assertTrue($postD->order === 5, 'D 5');
        $this->assertTrue($postD->group === 'base', 'D group');
        $this->assertTrue($postD->isHighestOrder, 'D self highest');
        $this->assertFalse($postD->isLowestOrder, 'D self lowest');
        $this->assertTrue($postF->order === 1, 'F 3');
    }
}
```
