```
<?php
use Illuminate\Support\Str;
use Illuminate\Support\Collection;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\Relation;

$morphMap = Collection::make(get_declared_classes())
   ->filter(function($class) use ($namespace) {
       return (
           Str::startsWith($class, $namespace) &&
           is_subclass_of($class, Model::class)
       );
   })
   ->map(function($class){
       return class_basename($class);
   })
   ->values();

Relation::morphMap($morphMap);

dd($morphMap);
```