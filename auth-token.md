# Built-in Token Auth


## URL
```
http://laravel.local/api/user?api_token=XXX.
```

## Api Route
```
Route::middleware('auth:api')->get('user', function(Request $request) {    
    return $request->user();
});
```

## Migration
```
$table->string('api_token', 60)->unique()->nullable()->default(null);
```

## Token Generator
```
<?php

do {
    $token = Str::random(128);
} while (User::where('api_token', $token)->exists());

$user = User::first($id)
$user->api_token = $token;
$user->save();

if($user = User::where('api_token', $token)->first()){
    //Do Something
}
```

## Axios Interceptor (using Quasar Cookies)
```
axios.interceptors.request.use((request) => {
    request.headers = {
        Authorization: 'Bearer ' + Cookies.get('api_token'),
    }
})
axios.interceptors.response.use((response) => {
    if(response.data.user){
        Cookies.set('api_token', response.data.user.api_token)
    }
})
```