# Magento Style 
Path-based lazy controller routing.  lol.

```php
<?php
Route::any("{controller}/{method}", "BaseController@callController");
```


```php
<?php
public function callController(string $controller, string $method)
{
  try{
    return app()->call("{$controller}@{$method}")
  }catch(\Throwable $e){
    abort(404);
  }
}
```
