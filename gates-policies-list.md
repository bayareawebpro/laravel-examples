# Gates & Policies List

```php
AppPermissions::all();
```
```php
[
  "ApiTokens" => [
    "apitokens:create",
    "apitokens:delete",
    "apitokens:forceDelete",
    "apitokens:restore",
    "apitokens:update",
    "apitokens:view",
    "apitokens:viewAny",
  ],
  "Attachments" => [
    "attachments:create",
    "attachments:delete",
    "attachments:forceDelete",
    "attachments:restore",
    "attachments:update",
    "attachments:view",
    "attachments:viewAny",
  ],
  "Users" => [
    "users:create",
    "users:delete",
    "users:forceDelete",
    "users:restore",
    "users:update",
    "users:view",
    "users:viewAny",
  ],
  "Abilities" => [
    "action:one",
    "action:two",
  ],
];
```
### AppPermissions Service Class

```php
use Illuminate\Support\Collection;

class AppPermissions{

    public static function all(): Collection
    {
       
        return Collection::make(Gate::policies())
            ->mapWithKeys(function (string $class) {
                
                $class = new ReflectionClass($class);
                $className= Str::plural(str_replace('Policy', '', $class->getShortName()));
                $classSlug= Str::lower($className);

                $list = Collection::make($class->getMethods())
                    ->pluck('name')
                    ->reject(fn($names)=>in_array($names,[
                        'allow', 'deny'
                    ]))
                    ->map(fn($method)=>"{$classSlug}:{$method}")
                    ->sort();

                return [$className => $list->values()];
            })
            ->sort()
            ->merge([
                'Abilities' => Collection::make(Gate::abilities())->keys()
            ]);
    }
}
```