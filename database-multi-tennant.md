```
<?php 
namespace App\Http\Middleware;

use Closure;
use Illuminate\Routing\Route;
use Illuminate\Support\Facades\DB;
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
	    config()->set('database.connections.tennant', [
		    'driver' => 'mysql',
		    'host' => '127.0.0.1',
		    'port' => '3306',
		    'database' => 'root',
		    'username' => 'root',
		    'password' => 'root',
		    'unix_socket' => '',
		    'charset' => 'utf8mb4',
		    'collation' => 'utf8mb4_unicode_ci',
		    'prefix' => '',
		    'strict' => true,
		    'engine' => null,
	    ]);
	    app('db')->setDefaultConnection('tennant');
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