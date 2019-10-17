```
$template = NestedValueResolver::resolveFirst($model, [
    'template',
    'category.template',
    'content_type.template',
]);
$templateArray = NestedValueResolver::resolveAll($model, [
    'template',
    'category.template',
    'content_type.template',
]);
```


```
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Collection;

class NestedValueResolver
{

    /**
     * Resolve the first Non-Empty Value from a Model using multiple DotSyntax keys.
     * @param Model $model
     * @param array $keys
     * @return mixed|null
     */
    public static function resolveFirst(Model $model, array $keys)
    {
        $result = null;
        Collection::make($keys)->each(function ($dotSyntax) use ($model, &$result) {
            return is_null($result = object_get($model, $dotSyntax, null));
        });
        return $result;
    }

    /**
     * Resolve many non-empty values from a Model using multiple DotSyntax keys.
     * @param Model $model
     * @param array $keys
     * @return mixed|null
     */
    public static function resolveAll(Model $model, array $keys)
    {
        return Collection::make($keys)
            ->transform(function ($dotSyntax) use ($model) {
                return object_get($model, $dotSyntax, null);
            })
            ->reject(function ($value) {
                return empty($value);
            })
            ->all();
    }
}

```