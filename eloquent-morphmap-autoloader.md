MorphMap AutoLoader
- https://implode.io/DybXb9

```
<?php
use Illuminate\Support\Str;
use Illuminate\Support\Collection;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\Relation;

$namespace = 'App\\Models';

$morphMap = Collection::make(get_declared_classes())
   ->filter(function($class) use ($namespace) {
       return (
           Str::startsWith($class, $namespace) &&
           is_subclass_of($class, Model::class)
       );
   })
   ->mapWithKeys(function($class){
       $mapName = Str::snake(Str::plural(class_basename($class)));
       return [$mapName => $class];
   })
   ->toArray();

Relation::morphMap($morphMap);

dd($morphMap);
```

```

use Illuminate\Database\Eloquent\Model;
class Post extends Model{
    //Bla bla bla...
}
class Category extends Model{
    //Bla bla bla...
}
class Tag extends Model{
    //Bla bla bla...
}
class Media extends Model{
    //Bla bla bla...
}
```