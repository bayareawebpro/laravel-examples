## CDN Resolver

```
<?php
declare(strict_types=1);

namespace App\Services;

class CdnResolver
{

    /**
     * Get Edge URL
     * @param $path
     * @param string $directory
     * @return \Illuminate\Contracts\Routing\UrlGenerator|string
     */
    public static function edge($path = '', $directory = 'media'): string
    {
        $host = config('filesystems.disks.spaces.edge');
        return url("{$host}/{$directory}/{$path}");
    }

    /**
     * Get Origin URL
     * @param $path
     * @param string $directory
     * @return \Illuminate\Contracts\Routing\UrlGenerator|string
     */
    public static function origin($path = '', $directory = 'media'): string
    {
        $host = config('filesystems.disks.spaces.origin');
        return url("{$host}/{$directory}/{$path}");
    }
}

```

## Helper Function
```
/**
 * @param $url
 * @param string $directory
 * @return \Illuminate\Contracts\Routing\UrlGenerator|string
 */
function cdnUrl($url, $directory = 'media'): string
{   
    return \App\Services\CdnResolver::edge($url, $directory);
}
```

## Filesystem Configuration
```
'disks' => [
       'spaces' => [
           'driver' => 's3',
           'key' => env('SPACES_KEY'),
           'secret' => env('SPACES_SECRET'),
           'endpoint' => env('SPACES_ENDPOINT'),
           'region' => env('SPACES_REGION'),
           'bucket' => env('SPACES_BUCKET'),
           'origin' => 'https://'.env('SPACES_BUCKET').'.'.env('SPACES_REGION').'.digitaloceanspaces.com',
           'edge' => 'https://'.env('SPACES_BUCKET').'.'.env('SPACES_REGION').'.cdn.digitaloceanspaces.com',
           'options' => [ 'CacheControl' => 'max-age=31536000, public' ]
       ],
   ...
}
```