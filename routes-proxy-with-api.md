
Add "api_token" field to your user model

- [how-to-use-laravels-built-in-token-auth](https://medium.com/@danielalvidrez/how-to-use-laravels-built-in-token-auth-6b6f6c26d059)
- http://laravel.local/api/resources/pages/1?api_token=XXX


```
<?php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\App;
Route::middleware('auth:api')->get('resources/{resource}/{page?}', function ($resource, $page = 1) {
   
    // Once authenticated using API, we swap the configured guards for the next call.
    // We need to use Web as the default guard to enable viaRemember checking
    // because it doesn't exist on the API/Token Guard.
    Auth::shouldUse('web');

    // We need to use the API/Token Guard for Nova instead of Web.
    Config::set('nova.guard', 'api');

    // Then we proxy the Request to the Nova Resource Controller.
    return App::handle(
        Request::create("nova-api/{$resource}", 'GET', [
            'page' => $page
        ])
    );
});

```