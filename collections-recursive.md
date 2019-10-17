```
Collection::macro('recursive', function(){
    return $this->map(function ($value) {
        if (is_array($value) || is_object($value)) {
            $collection = Collection::make($value);
            if(method_exists($collection, 'recursive')){
                return $collection->recursive();
            }
            return $collection;
        }
        return $value;
    });
});
```
