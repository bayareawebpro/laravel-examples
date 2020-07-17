# Data Collector Pattern

### Register Singleton Collection
Register in a service provider.
```php
$this->app->singleton('collector', fn()=>collect());
```

### Register Terminate Callback
Register in a service provider. Send a http request or store in the database.
```php
$this->app->terminating(function () {
    $data = $this->app->make('collector');
});
```

### Push values into the collector.
```php
app('collector')->push(['stats' => Str::random()]);
```