```
<?php
namespace App\Http\Middleware;
use Closure;
class RedirectRequestsToPWA
{
    /**
     * Handle an incoming request.
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
    	if(!$request->isRoute('pwa') && $request->method() === 'GET' && !$request->isXmlHttpRequest()){
		    return redirect(route('pwa').'/'.$request->path());
	    }
	    return $next($request);
    }
}
```