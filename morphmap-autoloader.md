```
<?php
$namespace = 'App\Models';
$models = array_filter(get_declared_classes(), function($index) use ($namespace) {
    return strpos($index, $namespace) === 0;
});
$morphMap = array();
foreach($models as $class){
    $mapName = snake_case(str_plural(class_basename($class)));
    $morphMap[$mapName] = $class;
}
\Illuminate\Database\Eloquent\Relations\Relation::morphMap($morphMap);
dd($morphMap);
```