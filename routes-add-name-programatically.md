```
Route::getFacadeRoot()
    ->getRoutes()
    ->getByAction('App\Http\Controllers\Auth\LoginController@login')
    ->name('login.submit');
```