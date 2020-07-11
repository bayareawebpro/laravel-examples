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

### Trait

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

### Service

```php
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Blade;
use Illuminate\Support\Str;
use Throwable;

class Mentions
{
    const MENTIONS_REGEX = "(@(?P<name>[a-zA-Z\-_]+))";

    private string $value;

    /**
     * Mentions constructor.
     * @param string $value
     */
    public function __construct(string $value)
    {
        $this->value = $value;
    }

    /**
     * Make Instance of Self.
     * @param string $value
     * @return static
     */
    public static function make(string $value): self
    {
        return app(static::class, ['value' => $value]);
    }

    /**
     * Get the mentions as a collection.
     * @return Collection
     */
    public function toCollection(): Collection
    {
        return Str::of($this->value)->matchAll(static::MENTIONS_REGEX);
    }

    /**
     * Compile the blade template for rendering.
     * @return string
     */
    public function compile(): string
    {
        return Blade::compileString($this->compileTemplate());
    }

    /**
     * Compile the string to a blade template.
     * @return string
     */
    protected function compileTemplate(): string
    {
        return (string) Str::of($this->value)->replaceMatches(static::MENTIONS_REGEX,
            fn($match) => $this->template($match['name'])
        );
    }

    /**
     * Wrap the name with the blade template tags.
     * @param $username
     * @return string
     * @throws Throwable
     */
    protected function template($username): string
    {
        return view('components.posts.mention', [
            'username' => $username
        ])->render();
    }
}

```
