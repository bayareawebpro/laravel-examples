# Vite HTTP Early Hints

```php
<?php declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Arr;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Config;
use Illuminate\Support\Facades\Vite;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class EarlyHintsMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        /**
         * @var $response \Illuminate\Http\Response
         */
        $response = $next($request);

        /**
         * Reject invalid methods and Content-Types.
         */
        if (!$this->isHtmlRequest($request, $response)) {
            return $response;
        }

        /**
         * Add CDN Domain for pre-connect.
         */
        $cdnDomain = Config::get('filesystems.disks.spaces.url');
        $response->header('Link', "<$cdnDomain>; rel=preconnect", false);

        /**
         * Add Vite Preloaded Assets.
         */
        foreach (Vite::preloadedAssets() as $url => $preload) {
            $directives = Collection::make($preload)->mapWithKeys(function (string $value) {
                $parts = Str::of($value)
                    ->remove('"')
                    ->explode('=');
                return [$parts[0] => $parts[1]];
            });
            $as = Arr::get($directives, 'as', 'script');
            $rel = Arr::get($directives, 'rel', 'preload');
            $rel = Str::remove('module', $rel);

            $response->header('Link', "<$url>; rel=$rel; as=$as", false);
        }
        
        return $response;
    }

    public function isHtmlRequest(Request $request, Response $response): bool
    {
        return ($request->isMethod('GET') && Str::contains($response->headers->get('Content-Type'), 'text/html'));
    }
}
```