# AB Test / Alternate View Resolver

Run AB test for user sessions where the views are swaped if they exist under a new path prefix.

```php
<?php declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Str;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Contracts\View\View;
use Illuminate\Contracts\View\Factory;

/**
 * Class ABTestResolver
 *
 * Start: /?redesign
 * Stop: /?redesign=false
 *
 * View: cms.content.single
 * Alt: redesign.content.single
 */
class ABTestResolver
{
    /**
     * View path prefix that will be prepended to the alternate view path.
     * @var string
     */
    protected string $pathPrefix = 'redesign';

    /**
     * Prefix or segments rejected from the view path.
     * @var array
     */
    protected array $rejectedPathSegments = [
        'cms',
    ];

    /**
     * Status codes allowed to the overridden during the test.
     * @var array
     */
    protected array $allowedStatusCodes = [
        200,
    ];


    /**
     * Handle an incoming request.
     * @param \Illuminate\Http\Request $request
     * @param \Closure $next
     * @return Response
     */
    public function handle(Request $request, Closure $next): Response
    {
        if ($this->userHasRequestedABTest($request)) {
            return $this->resolveAlternateView($next($request));
        }

        return $next($request);
    }

    /**
     * Verify the user has request the test.
     * @param Request $request
     * @return bool
     */
    protected function userHasRequestedABTest(Request $request): bool
    {
        if ($request->has('redesign')) {
            if ($request->filled('redesign')) {
                $request->session()->forget('redesign');
            } else {
                $request->session()->put('redesign', true);
            }
        }
        return $request->session()->has('redesign');
    }

    /**
     * Resolve the alternate view.
     * @param Response $response
     * @return Response
     */
    protected function resolveAlternateView(Response $response): Response
    {
        if (in_array($response->getStatusCode(), $this->allowedStatusCodes) && $response->original instanceof View) {

            $name = $this->formatViewPath($response->original->getName());

            /** @var $factory Factory */
            $factory = app(Factory::class);

            if ($factory->exists($name)) {
                $response->setContent($factory->make($name, $response->original->getData()));
            }
        }
        return $response;
    }

    /**
     * Format the overriding view path.
     * @param string $path
     * @return string
     */
    protected function formatViewPath(string $path): string
    {
        return Str::of($path)
            ->split('/\./')
            ->reject(fn($item) => in_array($item, $this->rejectedPathSegments))
            ->prepend($this->pathPrefix)
            ->join('.');
    }
}
```
