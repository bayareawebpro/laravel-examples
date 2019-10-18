### Usage

```
use Orderable;

// If you want to have groups of items that are ordered, then declare $orderGroup and specify which field
protected $orderGroup = ['parent_id'];
```

### Trait

```
<?php

namespace App\Traits;

use App\Menu;
use Illuminate\Support\Facades\DB;

trait Orderable
{
    public function getIsLowestOrderAttribute()
    {
        if (isset($orderGroup)) {
            $builder = static::where($orderGroup, $this[$orderGroup]);
        } else {
            $builder = static::all();
        }
        return $builder->min('order') === $this->order;
    }

    public function getIsHighestOrderAttribute()
    {
        if (isset($orderGroup)) {
            $builder = static::where($orderGroup, $this[$orderGroup]);
        } else {
            $builder = static::all();
        }
        return $builder->max('order') === $this->order;
    }

    public function getHighestOrderAttribute()
    {
        if (isset($orderGroup)) {
            $builder = static::where($orderGroup, $this[$orderGroup]);
        } else {
            $builder = static::query();
        }

        return $builder
            ->where('order', '>', $this->order)
            ->orderBy('order')
            ->first();
    }

    public function getLowestOrderAttribute()
    {
        if (isset($orderGroup)) {
            $builder = static::where($orderGroup, $this[$orderGroup]);
        } else {
            $builder = static::query();
        }

        return $builder
                ->where('order', '<', $this->order)
                ->orderBy('order', 'DESC')
                ->first();
    }

    public function incrementOrder()
    {
        if ($this->isHighestOrder) return;

        //Get the next highest order and decrease it.
        $next = $this->highestOrder;
        $next->order = $this->order;


        $this->order = $this->order+1;

        DB::transaction(function () use ($next) {
            $next->save();
            $this->save();
        });
    }

    public function decrementOrder()
    {
        //If menu is already min order, do nothing
        if ($this->isLowestOrder) return;


        //Get the next lowest order and increase it.
        $prev = $this->lowestOrder;

        $prev->order = $this->order;


        $this->order = $this->order-1;

        DB::transaction(function () use ($prev) {
            $prev->save();
            $this->save();
        });
    }
}
```