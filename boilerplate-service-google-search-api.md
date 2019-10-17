# Get Results
The service uses the global request with the option to overide parameters via an array.  Null / Empty / unset paramters will be removed.  Google's limit is 10 items per page max.  Caching is built in.
```
$this->api->getResults(array(
  'page' => 1,
  'search' => 10,
  'paginate' => 10,
  'site' => 'stackoverflow.com',
  'related => 'laravel.com',
  'terms' => 'auth,encryption',
  'images' => true,
  'filter' => 'i',
  'safe' => false
));
```

# Download Image
Utility method for working with image results.
```
$this->api->download('https://upload.wikimedia.org/wikipedia/commons/thumb/f/ff/DigitalOcean_logo.svg/1200px-DigitalOcean_logo.svg.png')
```

# Env & Service Config
```
GOOGLE_SEARCH_API=XX
GOOGLE_SEARCH_ENGINE=XX
GOOGLE_SEARCH_CACHE=false
GOOGLE_SEARCH_TTL=604800

config/services.php
'google' => [
        'search' => [
            'endpoint' => "https://www.googleapis.com/customsearch/v1",
            'token' => env('GOOGLE_SEARCH_API'),
            'engine' => env('GOOGLE_SEARCH_ENGINE'),
            'cache' => env('GOOGLE_SEARCH_CACHE', false),
            'cache_ttl' => env('GOOGLE_SEARCH_TTL', 604800),
        ],
    ],
```

# Controller
```
/**
 * GoogleController constructor.
 * @param GoogleSearch $api
 */
public function __construct(GoogleSearch $api)
{
    $this->api = $api;
}

/**
 * Get Results
 * @return \Illuminate\Http\JsonResponse
 * @throws mixed
 */
public function results()
{
    return response()->json($this->api->getResults());
}
```

# Service
```
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Cache;

use GuzzleHttp\Client;
use GuzzleHttp\Exception\GuzzleException;
use Illuminate\Support\Facades\Storage;

class GoogleSearch{
    protected $client;
    protected $request;
    protected $searchTerm;
    protected $results;
    protected $query;

    /**
     * GoogleController constructor.
     * @param Request $request
     * @param Client $client
     */
    public function __construct(Request $request, Client $client)
    {
        $this->client = $client;
        $this->request = $request;
        $this->results = new Collection;
        $this->query = new Collection;
    }

    /**
     * Make new instance of self.
     * @return GoogleSearch
     */
    public static function make(): GoogleSearch
    {
        return app(self::class);
    }

    /**
     * Get Query Params
     * @return Collection
     */
    protected function getQuery()
    {

        $page           = $this->request->get('page', 1);
        $perPage        = $this->request->get('paginate', 10);
        $searchTerm     = $this->request->get('search', null);
        $related        = $this->request->get('related', null);
        $terms          = $this->request->get('terms', null);
        $filter         = $this->request->get('filter', null);
        $site           = $this->request->get('site', null);
        $safe           = ($this->request->get('safe', false) === true) ? 'on' : 'off';
        $images         = ($this->request->get('images', false) === true) ? 'image' : null;

        $query = new Collection([
            'q'                 => $searchTerm,
            'relatedSite'       => $related,
            'siteSearch'        => $site,
            'siteSearchFilter'  => $filter,
            'searchType'        => $images,
            'exactTerms'        => $terms,
            'start'             => ($page * $perPage + 1) - $perPage,
            'num'               => $perPage < 10 ? $perPage : 10,
            'safe'              => $safe,
        ]);
        $query = $query->reject(function($item){
            return is_null($item) || empty($item);
        });

        logger('google:api:query', $query->toArray());

        return $query;

    }

    /**
     * Get Results
     * @param array $params
     * @return Collection
     * @throws mixed
     */
    public function getResults($params = array())
    {

        logger('google:api:params', $params);
        logger('google:api:request', $this->request->all());


        $this->request->merge($params);

        logger('google:api:merged', $this->request->all());

        $this->query    = $this->getQuery();
        $this->results  = new Collection($this->request($this->query));

        $items = new Collection($this->results->get('items'));

        logger('google:api:rawResults', $items->toArray());

        $this->results->put('items', $items->map(function ($result) {

            $thumb          = data_get($result, 'pagemap.cse_thumbnail.0.src');
            $image          = data_get($result, 'pagemap.cse_image.0.src', $thumb);
            $link           = data_get($result, 'link', '');
            $title          = data_get($result, 'title', '');
            $snippet        = data_get($result, 'snippet', '');
            $displayLink    = data_get($result, 'displayLink', '');

            return (object) array(
                'title'         => $title,
                'link'          => $link,
                'snippet'       => $snippet,
                'displayLink'   => $displayLink,
                'image'         => $this->query->has('searchType') ? $link : $image,
            );
        }));

        logger('google:api:transformedResults', $this->results->toArray());

        return $this->results;
    }

    /**
     * Perform Request
     * @param $query
     * @return Collection|mixed
     * @throws mixed
     */
    protected function request(Collection $query)
    {
        if(config('services.google.search.cache', false)){
            $cacheKey = "g-search-" . md5(http_build_query($query->toArray()));
            $cacheTtl = config('services.google.search.cache_ttl');
            return Cache::remember($cacheKey, $cacheTtl, function () use ($query){
                return $this->search($query);
            });
        }
        return $this->search($query);
    }

    /**
     * @param Collection $query
     * @return Collection
     * @throws mixed
     */
    protected function search(Collection $query)
    {
        $query->put('key', config('services.google.search.token'));
        $query->put('cx', config('services.google.search.engine'));

        try{
            $response = $this->client->request('GET', config('services.google.search.endpoint'), [
                'query' => $query->toArray()
            ]);
            $results = json_decode((string) $response->getBody(), true);
        }catch (GuzzleException $e){
            logger()->error($e->getMessage(), $e->getTrace());
            throw new \Exception('Invalid Parameters');
        }

        return new Collection($results);
    }

    /**
     * Download Remote Image
     * @param string $url
     * @return int
     */
    public function download($url)
    {
        try{

            Storage::disk('public')->makeDirectory('downloads');

            $file = fopen(storage_path('app/public/downloads/').basename($url),'w');

            /** @var $response \GuzzleHttp\Psr7\Response*/
            $response = $this->client->request('GET', $url, ['save_to' => $file]);

            fclose($file);

            return ($response->getStatusCode() === 200);

        } catch (GuzzleException $clientException) {

            logger()->error($clientException->getMessage());

            return false;

        }
    }
}


```