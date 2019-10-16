```
Collection::macro('recursive', function(){
    return $this->map(function ($value) {
        if (is_array($value) || is_object($value)) {
            $collection = collect($value);
            if(method_exists($collection, 'recursive')){
                return $collection->recursive();
            }
            return $collection;
        }
        return $value;
    });
});
```