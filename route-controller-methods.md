# Magento Style 
Path-based lazy controller routing.  lol.

```php
<?php
Route::any("{controller}/{method}", "BaseController@callController");
```


```php
<?php
public function callController($controller, $method)
{
  try{
    return app()->call("{$controller}@{$method}")
  }catch(\Throwable $e){
    abort(404);
  }
}
```
