# Larave GeoCoder (GeoIP)

## Usage

```php
<?php

$result = Geocoder::using('chain')
	->geocode("$lat, $lon")
	->get()
	->first();

$result = Geocoder::using('geoip2')
	->geocode($ipAddress)
	->get()
	->first();
```

## Dependencies
- toin0u/geocoder-laravel
- geocoder-php/geoip2-provider
- php-http/guzzle6-adapter
- guzzlehttp/guzzle

      
## Config
```php
<?php

use Geocoder\Provider\Chain\Chain;
use Geocoder\Provider\GeoIP2\GeoIP2;
use Geocoder\Provider\GeoPlugin\GeoPlugin;
use Geocoder\Provider\GoogleMaps\GoogleMaps;

return [
    'cache'     => [

        /*
        |-----------------------------------------------------------------------
        | Cache Store
        |-----------------------------------------------------------------------
        | Specify the cache store to use for caching. The value "null" will use
        | the default cache store specified in /config/cache.php file.
        | Default: null
        */
        'store'    => 'file',

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
        Chain::class  => [
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
    'adapter'   => Http\Adapter\Guzzle6\Client::class,

    /*
    |---------------------------------------------------------------------------
    | Database Reader
    |---------------------------------------------------------------------------
    | You can specify a reader for specific providers, like GeoIp2, which
    | connect to a local file-database. The reader should be set to an
    | instance of the required reader class.
    | Please consult the official Geocode documentation for more info.
    | https://github.com/geocoder-php/geoip2-provider
	*/
    'reader'    => [
        \GeoIp2\Database\Reader::class => [
            database_path('maxmind/GeoLite2-City.mmdb'),
        ],
    ],
];
```
