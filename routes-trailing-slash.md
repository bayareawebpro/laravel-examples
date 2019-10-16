## Implementation

```

$this->app->singleton('url', function (Application $app) {

    //Set the Session Resolver for the UrlGenerator.
    $routes = $app->instance('routes', $app['router']->getRoutes());

    //Create the UrlGenerator Instance.
    $url = new UrlGenerator($routes, $app['request']);

    //Set the Session Resolver for the UrlGenerator.
    $url->setSessionResolver(function () use ($app) {
        return $app['session'];
    });

    //Set the Key Resolver for the UrlGenerator.
    $url->setKeyResolver(function () use ($app) {
        return $app['config']->get('app.key');
    });

    //Sync Routes with UrlGenerator when ReBound Event Fires.
    $app->rebinding('routes', function ($app, $routes) {
        $app['url']->setRoutes($routes);
    });

    return $url;
});
```

## URL Generator Replacement
```
<?php
namespace App\Services;
use Illuminate\Support\Str;
use Illuminate\Routing\UrlGenerator as BaseUrlGenerator;
class UrlGenerator extends BaseUrlGenerator
{
    /**
     * Format the given URL segments into a single URL.
     * @param  string  $root
     * @param  string  $path
     * @param  \Illuminate\Routing\Route|null  $route
     * @return string
     */
    public function format($root, $path, $route = null)
    {
        return $this->addTrailingSlash(parent::format($root, $path, $route));
    }

    /**
     * Generate an absolute URL to the given path.
     * @param  string  $path
     * @param  mixed  $extra
     * @param  bool|null  $secure
     * @return string
     */
    public function to($path, $extra = [], $secure = null)
    {
        //Filter Valid URLs Always
        if ($this->isValidUrl($path)) {
            return $this->addTrailingSlash($path);
        }
        return parent::to($path, $extra, $secure);
    }

    /**
     * Add Trailing Slash to Urls (Used In Pagination)
     * @param $url string
     * @return string
     */
    public function addTrailingSlash($url)
    {
        if(!pathinfo(parse_url($url, PHP_URL_PATH), PATHINFO_EXTENSION)){
            if(Str::contains($url, '?')){
                if(!Str::contains($url, '/?')) {
                    $url = Str::replaceFirst('?', "/?", $url);
                }
            }elseif (!Str::endsWith($url, '/')) {
                $url = "$url/";
            }
        }
        return $url;
    }
}
```


## Middleware 

```
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Str;

class RedirectToTrailingSlash
{
    /**
     * Handle an incoming request.
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if(!Str::endsWith($request->getPathInfo(), ['/'])){
            return redirect(url($request->url()), 302);
        }
        return $next($request);
    }
}
```

## Test Case Override:
```
/**
 * Turn the given URI into a fully qualified URL.
 * @param  string  $uri
 * @return string
 */
protected function prepareUrlForRequest($uri)
{
    if (! Str::startsWith($uri, 'http')) {
        $uri = config('app.url').'/'.$uri;
    }
    return $uri;
}
```

## Test Cases:
```
<?php namespace Tests\Unit;

use App\Page;
use Tests\TestCase;

class TrailingSlashTest extends TestCase
{
    public function test_it_can_append_to_domain()
    {
        $this->assertSame(
            'https://google.com/',
            url('https://google.com'),
            'Append to domain.'
        );
    }

    public function test_will_ignore_extensions()
    {
        $this->assertSame(
            'https://google.com/image.jpg',
            url('https://google.com/image.jpg'),
            'Ignore extensions.'
        );
    }

    public function test_will_append_to_path()
    {
        $this->assertSame(
            'https://google.com/segment/',
            url('https://google.com/segment'),
            'Append to path.'
        );
    }

    public function test_will_append_before_query()
    {
        $this->assertSame(
            'https://google.com/?prop=1&prop=2',
            url('https://google.com?prop=1&prop=2'),
            'Append before query.'
        );
    }

    public function test_will_append_before_query_after_segments()
    {
        $this->assertSame(
            'https://google.com/segment1/segment2/?prop=1&prop=2',
            url('https://google.com/segment1/segment2?prop=1&prop=2'),
            'Append before query after segment.'
        );
    }

    public function test_will_append_append_to_empty_url()
    {
        $this->assertSame(
            config('app.url').'/',
            url(config('app.url'))
        );
    }
}

```