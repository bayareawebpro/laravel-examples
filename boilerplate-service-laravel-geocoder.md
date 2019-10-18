## Usage

```
use App\Services\GeocodeApi;
$result = GeocodeApi::make()->geocode($searchTerm);
$result = GeocodeApi::make()->getCountryCode($fallback = 'US');
$result = app(GeocodeApi::class)->getCountryCode($fallback = 'US');
```

## Dependencies
- toin0u/geocoder-laravel
- php-http/guzzle6-adapter
- guzzlehttp/guzzle
- geocoder-php/geoip2-provider
- https://github.com/bayareawebpro/laravel-examples/blob/master/collections-recursive.md
        
## Service
```
<?php declare(strict_types=1);

namespace App\Services;

use GeoIp2\Database\Reader;
use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Illuminate\Config\Repository as Config;
use Geocoder\Provider\GoogleMaps\Model\GoogleAddress;

class GeocodeApi
{
    protected $request, $cache, $config;

    /**
     * GeocodeApi constructor.
     * @param Request $request
     * @param Config $config
     * @throws \Exception
     */
    public function __construct(Request $request, Config $config){
        $this->request = $request;
        $config->set('geocoder.reader', new Reader(database_path('maxmind/GeoLite2-City.mmdb')));
    }

    /**
     * Make new instance of self.
     * @return GeocodeApi
     */
    public static function make(): GeocodeApi
    {
        return app(self::class);
    }

    /**
     * Geocode String using Service
     * @param string $searchTerm
     * @param string $service
     * @return array|null
     */
    public function geocode($searchTerm = 'Santa Cruz, CA', $service = 'chain')
    {
        $result = app('geocoder')
            ->using($service)
            ->geocode($searchTerm)
            ->get()
            ->first();

        /** @var $result GoogleAddress */
        if(!is_null($result)) {

            /** @var $location Collection */
            $location = Collection::make($result->toArray())->recursive();

            $address = new Collection();
            $address->put('city',  $location->get('locality') ?? $location->get('subLocality'));
            $address->put('county', null);
            $address->put('state', null);
            $address->put('zip',  $location->get('postalCode', null));
            $address->put('latitude',  $location->get('latitude', null));
            $address->put('longitude',  $location->get('longitude', null));
            $address->put('country',  $location->get('country', null));
            $address->put('iso',  $location->get('countryCode', null));

            if($levels = $location->get('adminLevels', false)){
                if($county = $levels->where('level', 2)->first()){
                    $address->put('county', data_get($county, 'name'));
                }
                if($state = $levels->where('level', 1)->first()){
                    $address->put('state', data_get($state, 'name'));
                    $address->put('state_code', data_get($state, 'code'));
                }
            }

            $address = $address->reject(function($value){
                return is_null($value);
            });
            return $address->toArray();
        }
        return $result;
    }

    /**
     * Get Country Code from IP Address
     * @param null $ipAddress
     * @param string $fallback
     * @return mixed|string
     */
    public function getCountryCode($ipAddress = null, $fallback = 'US')
    {
        if (is_null($ipAddress)) {
            $ipAddress = $this->request->ip();
        }
        //Get the result from the API Request.
        if (!empty($ipAddress)) {
            $result = $this->geocode($ipAddress, 'geoip2');
            if ($result) {
                return data_get($result, 'iso', $fallback);
            }
        }
        return $fallback;
    }
}

```

## Config
```
<?php
use Geocoder\Provider\Chain\Chain;
use Geocoder\Provider\GeoIP2\GeoIP2;
use Geocoder\Provider\GeoPlugin\GeoPlugin;
use Geocoder\Provider\GoogleMaps\GoogleMaps;
return [
    'cache' => [

        /*
        |-----------------------------------------------------------------------
        | Cache Store
        |-----------------------------------------------------------------------
        | Specify the cache store to use for caching. The value "null" will use
        | the default cache store specified in /config/cache.php file.
        | Default: null
        */
        'store' => 'file',

        /*
        |-----------------------------------------------------------------------
        | Cache Duration
        |-----------------------------------------------------------------------
        | Specify the cache duration in minutes. The default approximates a
        | "forever" cache, but there are certain issues with Laravel's forever
        | caching methods that prevent us from using them in this project.
        | Default: 9999999 (integer)
        |
        */
        'duration' => PHP_INT_MAX,
    ],

    /*
    |---------------------------------------------------------------------------
    | Providers
    |---------------------------------------------------------------------------
    | Here you may specify any number of providers that should be used to
    | perform geocaching operations. The `chain` provider is special,
    | in that it can contain multiple providers that will be run in
    | the sequence listed, should the previous provider fail. By
    | default the first provider listed will be used, but you
    | can explicitly call subsequently listed providers by
    | alias: `app('geocoder')->using('google_maps')`.
    | Please consult the official Geocode documentation for more info.
    | https://github.com/geocoder-php/Geocoder#providers
    */
    'providers' => [
        GeoIP2::class => [],
        Chain::class => [
            GoogleMaps::class => ['en-US', env('GOOGLE_SECRET')],
        ],
        //GeoPlugin::class  => [],
    ],
    /*
    |---------------------------------------------------------------------------
    | Adapter
    |---------------------------------------------------------------------------
    | You can specify which PSR-7-compliant HTTP adapter you would like to use.
    | There are multiple options at your disposal: CURL, Guzzle, and others.
    | Please consult the official Geocode documentation for more info.
    | https://github.com/geocoder-php/Geocoder#usage
    | Default: Client::class (FQCN for CURL adapter)
    */
    'adapter'  => Http\Adapter\Guzzle6\Client::class,

    /*
    |---------------------------------------------------------------------------
    | Reader (Set in App Service Provider upon Boot.)
    |---------------------------------------------------------------------------
    | You can specify a reader for specific providers, like GeoIp2, which
    | connect to a local file-database. The reader should be set to an
    | instance of the required reader class.
    | Please consult the official Geocode documentation for more info.
    | https://github.com/geocoder-php/geoip2-provider
	|-----------------------------------------------------------------------
	| MaxMind Database Reader
	|-----------------------------------------------------------------------
	| Default: null
	*/
    'reader' => null, //Set by service class when used, cannot serialize instance.
];
```