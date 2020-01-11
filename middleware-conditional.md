> Notice: This example needs further troubleshooting for the newest verson 6.x.  Use with caution.

```php
<?php
namespace App\Http\Middleware;

use Closure;

class ConditionalMiddlware
{
    /**
     * Handle an incoming request.
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if(app()->environment('staging', 'production')){
            app('router')->pushMiddlewareToGroup('web','throttle:75,1');
        }
        return $next($request);
    }
}

```
