## Trait

```
<?php declare(strict_types=1);

namespace App\Models\Traits;

use App\Models\User;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

trait OwnableTrait{

    /**
     * Owned By Ownable
     * @param Builder $query
     * @param User $user
     * @return Builder
     */
    public function scopeOwnedBy(Builder $query, ?User $user): Builder
    {
        return $query->where('user_id', $user->id);
    }

    /**
     * Belongs to User
     * @return BelongsTo
     */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

## Interface

```
<?php declare(strict_types=1);

namespace App\Models\Traits;
use App\Models\User;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

interface Ownable{

    /**
     * Owned By Scope
     * @param Builder $query
     * @param User $user
     * @return Builder
     */
    public function scopeOwnedBy(Builder $query, ?User $user): Builder;

    /**
     * Belongs to User
     * @return BelongsTo
     */
    public function user(): BelongsTo;
}

```
