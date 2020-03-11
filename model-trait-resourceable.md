# Resourceable Trait

- Contributor (via gist): https://github.com/christopherarter
- Refactoring: https://github.com/bayareawebpro

### Default namespace guesser: 

> \MyApp\Http\Resources\UserResource::class
>
```php
$user = new User([...]);
return $user->toResourceArray(); 
```

### Using a custom namespaced class.

```php
$user = new User([...]);
return $user->toResourceArray(ListResource::class);
```


```php
<?php

namespace App\Concerns;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

trait Resourceable
{
    public function toResourceArray(?string $class = null, ?Request $request = null): array
    {
        $class = $class ?: $this->resolveResourceName();

        if (class_exists($class) && is_subclass_of($class, JsonResource::class)) {
            return (new $class($this))->resolve($request);
        }

        return $this->toArray();
    }

    protected function resolveResourceName(): string
    {
        $reflect = new \ReflectionClass($this);
        return "\\{$reflect->getNamespaceName()}\\Http\\Resources\\{$reflect->getShortName()}Resource";
    }
}
```