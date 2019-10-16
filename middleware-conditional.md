```
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Routing\Router;
use Illuminate\Support\Str;

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