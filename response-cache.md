## Response Cache Service / Middleware

- Similar to Spatie's Response Cache but will minify the html before caching. (see related service)
- Able to prime the cache via jobs using the public methods.

#### Usage: 
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\Config;

class ResponseCache
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if(Config::get('response.cache.enabled', false)){
            return \App\Services\ResponseCache::make()->handle($next);
        }

        return $next($request);
    }
}

```

#### Service: 
```php
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Http\Request;
use Illuminate\Cache\Repository;
use Symfony\Component\HttpFoundation\Response;

class ResponseCache
{
    protected $request;
    protected $cache;

    public function __construct(Repository $cache, Request $request)
    {
        $this->cache = $cache->tags(config('response.cache.tags', ['response']));
        $this->request = $request;
    }

    public static function make(): ResponseCache
    {
        return app(static::class);
    }

    public function handle(\Closure $next)
    {
        if ($this->isCached()) {
            return $this->restoreFromCache();
        }

        /** @var Response $response */
        $response = $next($this->request);

        if ($this->shouldCache($response)) {
            $this->storeInCache($response);
        }

        return $response;
    }

    public function storeInCache(Response $response): bool
    {
        $content = $response->getContent();
        $content = $this->replaceToken($content);
        $content = $this->minifyContent($content);

        return $this->cache->forever($this->getCacheKey(), [
            'headers' => $response->headers->all(),
            'status'  => $response->getStatusCode(),
            'content' => $content,
        ]);
    }

    public function restoreFromCache(): Response
    {
        $data = $this->cache->get($this->getCacheKey());

        $data['content'] = $this->restoreToken($data['content']);
        $data['headers'] = array_merge($data['headers'], [
            'X-Cached' => now()->toRfc2822String(),
        ]);

        return response(
            $data['content'],
            $data['status'],
            $data['headers']
        );
    }

    protected function shouldCache(Response $response): bool
    {
        return (
            $this->request->isMethod('GET') &&
            is_string($response->getContent()) &&
            $response->isSuccessful()
        );
    }

    protected function replaceToken(string $content): string
    {
        return str_replace(csrf_token(), config('response.cache.token_placeholder', '<TOKEN>'), $content);
    }

    protected function restoreToken(string $content): string
    {
        return str_replace(config('response.cache.token_placeholder', '<TOKEN>'), csrf_token(), $content);
    }

    protected function minifyContent(string $content): string
    {
        return HtmlMinifier::minify($content);
    }

    protected function getCacheKey(): string
    {
        return "response-" . md5($this->request->getRequestUri());
    }

    protected function isCached(): bool
    {
        return $this->cache->has($this->getCacheKey());
    }

    public function flushCache(): void
    {
        $this->cache->clear();
    }
}
```

#### Example Job: Cache Primer 
```php
<?php declare(strict_types=1);

namespace App\Jobs;

use App\Services\ResponseCache;
use App\Services\ContentResolver;

use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Laravel\Telescope\Telescope;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Bus\Queueable;

class PrimeCacheForPages implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * @var Collection
     */
    public $slugs;

    /**
     * Create a new job instance.
     * @param Collection $slugs
     */
    public function __construct(Collection $slugs)
    {
        $this->slugs = $slugs;
    }

    /**
     * Execute the job.
     * @return void
     */
    public function handle()
    {
        Telescope::stopRecording();

        $this->slugs->each(function($slug){

            /**
            * Make the cache service.
            * We are running in "console context" and we will need a 
            * "Faux Request" instance for the url cache key.
            */
            $cache = app(ResponseCache::class, ['request' => Request::create(url($slug))]);
            
            // Get the controller's view response programmatically.
            $response = app()->call('\App\Http\Controllers\PageController@show', ['page' => $slug]);

            // Store the response in the cache.
            $cache->storeInCache($response);
        });
    }
}
```