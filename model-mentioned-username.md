# Username Mentions

### Usage

```php
$post = new Post($request->validated());
$post->user()->associate($request->user());
$post->save();
$post->getMentionedUsers()->each(fn($user)=>$user->notify(new MentionNotification($post)));
```

### Configure Model

```php
use App\Concerns\HasMentions;

class Post extends Model
{
    use HasMentions;
    
    /**
     * Parse Mentions from Attribute
     * @var string
     */
    public string $mentionable = 'content';
```

```php
<?php declare(strict_types=1);

namespace App\Concerns;

use App\User;
use App\Services\Mentions;
use Illuminate\Database\Eloquent\Collection as EloquentCollection;
use Illuminate\Support\Collection;

trait HasMentions
{

    //public string $mentionable = 'content';

    /**
     * Compile Username Mentions to Raw Blade Syntax.
     * @return string
     */
    public function compileMentions(): string
    {
        return Mentions::make(nl2br($this->getAttribute($this->mentionable)))->compile();
    }

    /**
     * Get Username Mentions Collection.
     * @return Collection
     */
    public function getMentions(): Collection
    {
        return Mentions::make($this->getAttribute($this->mentionable))->toCollection();
    }

    /**
     * Find Mentioned Users by Username.
     * @return EloquentCollection
     */
    public function getMentionedUsers(): EloquentCollection
    {
        return User::findByUsername(...$this->getMentions()->take(100)->toArray());
    }
}
```
