## Invokable Queries

#### Usage

```php

// Array Spread / Merge Validation Rules
$request->validate([
    ...NameQuery::rules(),
    ...RoleQuery::rules(),
    ...EmailQuery::rules(),
]);

// Apply Conditionally
User::query()
    ->when(NameQuery::isRelevant($request), NameQuery::make($request))
    ->when(RoleQuery::isRelevant($request), RoleQuery::make($request))
    ->when(EmailQuery::isRelevant($request), EmailQuery::make($request))
    ->paginate(25)
;

// Apply Directly
User::query()
    ->tap(NameQuery::make($request))
    ->tap(RoleQuery::make($request))
    ->tap(EmailQuery::make($request))
    ->paginate(25)
;
```

#### Contract

```php
<?php namespace App\Queries\Contracts;

use Illuminate\Http\Request;
use Illuminate\Database\Eloquent\Builder;

interface InvokableQuery
{
    public function __construct(Request $request);
    public function __invoke(Builder $builder): void;
    public static function make(Request $request): self;
    public static function isRelevant(Request $request): bool;
    public static function rules(?Request $request = null): array;
}

```

#### Implementation

```php
<?php declare(strict_types=1);

namespace App\Queries;

use Illuminate\Http\Request;
use Illuminate\Database\Eloquent\Builder;
use App\Queries\Contracts\InvokableQuery;

class EmailQuery implements InvokableQuery
{
    protected Request $request;
    protected static string $field = 'email';
    protected static string $attribute = 'email';

    public static function make(Request $request): self
    {
        return new static($request);
    }

    public function __construct(Request $request)
    {
        $this->request = $request;
    }

    public function __invoke(Builder $builder): void
    {
        $builder->where(static::$attribute, $this->request->get(static::$field));
    }

    public static function isRelevant(Request $request): bool
    {
        return $request->filled(static::$field);
    }

    public static function rules(?Request $request = null): array
    {
        return [
            [static::$field => 'sometimes|string|email|max:255'],
        ];
    }
}
```
