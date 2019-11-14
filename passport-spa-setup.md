### Installation

- Automatic Token Auth, after Log-In via Web Auth.
- No Tokens Stored in LocalStorage.
- No Tokens Stored in Database.
- No OAuth Needed.

```shell script
composer require laravel/passport
artisan passport:install
artisan migrate
```

### Configuration

**config/auth.php**

```php
<?php
return [
    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
        'api' => [
            'driver' => 'passport',
            'provider' => 'users',
            'hash' => false,
        ],
    ],
];
```

### Web Routes
```php
<?php
Auth::routes();
Route::view('{slug?}', 'layouts.spa')
    ->where('slug', '^(?!telescope|nova).*?$')
    ->name('spa');
```

### Api Route
```php
<?php
Route::group([
    'prefix' => 'account',
    'middleware' => 'auth:api',
], function(){
    Route::get('/', function(){
        return response(['user' => request()->user()]);
    });
});
```

### Update RedirectIfAuthenticated Middleware
We'll return our user for json requests.

```php
<?php
if (Auth::guard($guard)->check()) {
    if($request->wantsJson()){
        return response()->json(['user' => $request->user()]);
    }
    return redirect('/');
}
```


### Add CreateFreshApiToken Middleware
This is where the magic happens, everytime we visit a web route, we get a new token when authorized. So, 
once we login, we're ready to use authenticated API routes. Laravel will inject a cookie into web route 
responses that will be automatically used by Axios during future api requests. We use an HttpOnly cookie to
 secure our token. No need to store anything manually.

```php
<?php
$middlewareGroups = [
    'web' => [
        //...
        \Laravel\Passport\Http\Middleware\CreateFreshApiToken::class,
    ],
    'api' => [
        //...
    ],
];
```

### Now, for some additional enhancements:
If we want our login and logout functionality to be seamless we need to do a few things to give Axios some help.

### Login & Logout CSRF Handling
Override the trait methods to return your user and the csrf_token.
(App\Http\Controllers\Auth\LoginController.php)

```php
<?php
/**
 * The user has been authenticated.
 * @param  \Illuminate\Http\Request  $request
 * @param  mixed  $user
 * @return mixed
 */
protected function authenticated(Request $request, $user)
{
    return response()->json([
        'user' => $user,
        'csrf_token' => $request->session()->token()
    ]);
}

/**
 * The user has logged out of the application.
 * @param  \Illuminate\Http\Request  $request
 * @return mixed
 */
protected function loggedOut(Request $request)
{
    $request->session()->regenerate();
    return response()->json([
        'csrf_token' => $request->session()->token()
    ]);
}
```

### Axios Interceptor
```javascript
client.interceptors.response.use((response) => {
    if(response.data && response.data.csrf_token){
        client.defaults.headers.common['X-CSRF-TOKEN'] = csrf_token
    }
    return Promise.resolve(response)
})
```
