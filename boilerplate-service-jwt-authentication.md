# Json Web Token (JWT) Authentication

### Register in Auth Service Provider.

```php
JsonWebToken::model(User::class, 'token');
```

### Configure Auth.php
```
'api' => [
    'driver' => 'laravel-jwt',
    'provider' => 'users',
    'hash' => false,
],
```

### Create New Token
```php
$token = JsonWebToken::createTokenForUser(User::first(), now()->addHours(3), [
  'my_key' => true
]);
```

### Authenticate
```text
http://laravel.test/api/user?token=xxx
```

### Get Data From Token
```php
$request->jwt()->get('my_key');
```


### Service Class
```php
<?php declare(strict_types=1);

namespace App\Services;

use RuntimeException;
use Throwable;
use Carbon\Carbon;
use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Crypt;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Auth\Access\AuthorizationException;

class JsonWebToken
{
    public static string $model;
    public static string $decoded;

    /**
     * @param string $model
     * @param string $keyName
     */
    public static function model(string $model, string $keyName = 'token'): void
    {
        Auth::viaRequest('laravel-jwt', new static);
        Request::macro('jwt', function() use ($keyName){
            if(app()->bound('jwt-decoded')){
                return app()->get('jwt-decoded');
            }
            return app()->instance('jwt-decoded', JsonWebToken::parseToken($this->get($keyName)));
        });
        static::$model = $model;
    }

    /**
     * Invoke the authorization resolver.
     * @param Request $request
     * @return mixed
     */
    public function __invoke(Request $request)
    {
        try {
            $token = $request->jwt();
            throw_unless($this->isValidTimestamp($token->get('valid')), AuthorizationException::class);
            return static::$model::query()->find($token->get('user'));
        } catch (Throwable $e) {
            logger()->error($e->getMessage(), $e->getTrace());
        }
    }

    /**
     * Create a new token for a model.
     * @param Model $user
     * @param array $data
     * @param Carbon|null $until
     * @return string
     */
    public static function createTokenForUser(Model $user, ?Carbon $until = null,array $data = []): string
    {
        $until = ($until ?? Carbon::now()->addDays(30))->toDateTimeString();
        return Crypt::encryptString(
            Collection::make(['valid' => $until, 'user'  => $user->getKey()])
            ->merge($data)
            ->toJson()
        );
    }

    /**
     * Parse the token to collection instance.
     * @param string $token
     * @return Collection
     * @throws RuntimeException
     */
    public static function parseToken(string $token): Collection
    {
        return Collection::make(json_decode(Crypt::decryptString($token)));
    }

    /**
     * Is the timestamp claim valid.
     * @param string $timestamp
     * @return bool
     */
    public function isValidTimestamp(string $timestamp): bool
    {
        return Carbon::parse($timestamp)->greaterThanOrEqualTo(now());
    }
}

```
