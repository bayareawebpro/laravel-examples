```
<?php 
namespace App\Http\Middleware;

use Closure;
use Illuminate\Routing\Route;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Config;
class Tenant
{
    /**
     * Handle an incoming request.
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        // Get Configuration for Tennant based on MySql Connection
        $config = Collection::make(Config::get('database.connections.mysql'));

        // Get Subdomain as name (fetch model) ?
        $tennentName = $request->route('subdomain');

        // Set Credentials 
        $config->put('database', 'my-tennant');
        $config->put('username', 'my-tennant');
        $config->put('password', 'my-tennant');
        
        // Set New Connection 
	    Config::set('database.connections.tennant',$config->toArray());

        // Use New Connection 
	    DB::setDefaultConnection('tennant');

	    return $next($request);
    }
}

```

```
/**
 * The application's route middleware groups.
 * @var array
 */
protected $middlewareGroups = [
    'api' => [
        Tennant::class,
    ],
];
```

```
Route::domain('{subdomain}.artisan.local')->group(function(){
	// Tennant Routes
});
```