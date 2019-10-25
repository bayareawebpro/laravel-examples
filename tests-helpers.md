#### With Session
Override the parent method, sets session on request correctly for unit tests.
```php
public function withSession(array $data = [])
{
    $this->session($data);
    Request::setLaravelSession($this->app['session']);
}
```

#### Telescope

**Start**
```php
protected function telescopeStart()
{
    Telescope::startRecording();
}
```

**Stop**
```php
protected function telescopeStop()
{
    Telescope::stopRecording();
}
```

**Clear**
```php
protected function telescopeClear()
{
    app(ClearableRepository::class)->clear();
}
```

#### With Routes

```php
public function withRoutes(\Closure $callback)
{
    $router = app('router');
    $callback($router);
    $router->getRoutes()->refreshNameLookups();
}
```

Add within test methods easily.
```php
$this->withRoutes(function(\Illuminate\Routing\Router $router){
    $router->get('/test-route/', function(){
        return response('ok');
    })->name('test-route');
});
dd($this->get(route('test-route'))->getContent());
```

Insure it doesn't interfere with any wildcard routes.
```php
Route::get("{slug}", 'PageController@show')
    ->where('slug', '^(?!test-|telescope|nova|api)([^.]*)$')
    ->name("page");
```

#### Faker
```php
protected function faker()
{
    return app(\Faker\Generator::class);
}
```

#### Ajax Headers
```php
public function ajaxHeaders(): array
{
    return [
        'X-Requested-With' => "XMLHttpRequest",
        'Accept'           => "application/json",
        'X-CSRF-TOKEN'     => csrf_token(),
    ];
}
```

#### Bypass Assertions if config not set.
Usage: 
```php
if($this->doesntAssertIfEnvNotConfigured()) return;
$this->assertTrue(false);
```

Environment Config:
```php
<?php
return [
    'shouldAssert'=>'0.0.0.0:32776'
];
```
Helper: 
```php
protected function doesntAssertIfEnvNotConfigured($configKey = null)
{
    if(!config()->has($configKey ?? 'ssh.host')){
        $this->expectNotToPerformAssertions();
    }
}
```

#### Environment Setup
```php
protected function getEnvironmentSetUp($app)
{
    $env = __DIR__.'/../.env.php';
    if(file_exists($env)){
        $app['config']->set('ssh', require __DIR__.'/../.env.php');
    }else{
        $app['config']->set('ssh', []);
    }
}
```