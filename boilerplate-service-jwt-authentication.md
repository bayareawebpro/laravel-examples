# Json Web Token (JWT) Authentication

### Register in Auth Service Provider.

```php
JsonWebToken::register(User::class, 'token');
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
$request->jwt('my_key');
```


### Service Class
```php
<?php declare(strict_types=1);

namespace App\Services;

use Throwable;
use Carbon\Carbon;
use RuntimeException;
use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Crypt;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Auth\Access\AuthorizationException;
use Illuminate\Contracts\Encryption\DecryptException;

class JsonWebToken
{
    public static string $model;
    public static string $decoded;

    /**
     * @param string $model
     * @param string $keyName
     */
    public static function register(string $model, string $keyName = 'token'): void
    {
        static::$model = $model;

        Auth::viaRequest('laravel-jwt', new static);

        Request::macro('jwt', function(?string $key = null) use ($keyName){
            $token = JsonWebToken::singletonInstance($this->get($keyName));
            if($key){
                return $token->get($key);
            }
            return $token;
        });
    }

    /**
     * Get Singleton Instance of Token
     * @param $token
     * @return Collection
     */
    public static function singletonInstance($token): Collection
    {
        $app = app();
        return $app->bound($token)
            ? $app->get($token)
            : $app->instance($token, JsonWebToken::parseToken($token));
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
            throw_unless($token->get('valid'), AuthorizationException::class);
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
            Collection::make(['expires' => $until, 'user'  => $user->getKey()])
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
        try{
            $token = Collection::make(json_decode(Crypt::decryptString($token)));
            $token->put('valid', static::isValidTimestamp($token->get('expires')));
            return $token;
        }catch (DecryptException $exception){
            return Collection::make(['valid'=> false]);
        }
    }

    /**
     * Is the timestamp claim valid.
     * @param string $timestamp
     * @return bool
     */
    public static function isValidTimestamp(string $timestamp): bool
    {
        return Carbon::parse($timestamp)->greaterThanOrEqualTo(now());
    }
}
```

### Unit Tests

```php
<?php namespace Tests\Feature;

use App\User;
use Tests\TestCase;
use App\Services\JsonWebToken;

class JwtTest extends TestCase
{
    public function test_cannot_authorize_user()
    {
        $this
            ->json('GET',"/api/user?token=fake")
            ->assertStatus(401)
        ;
    }

    public function test_can_authorize_user()
    {
        $user = factory(User::class)->create();
        $token = JsonWebToken::createTokenForUser($user);

        $this
            ->json('GET',"/api/user?token={$token}")
            ->assertStatus(200)
            ->assertJson($user->toArray())
        ;
    }

    public function test_valid_token()
    {
        $token = JsonWebToken::parseToken(JsonWebToken::createTokenForUser(
            factory(User::class)->create(),
            now()->addRealSeconds(60), [
            'test' => 123
        ]));
        $this->assertTrue($token->get('valid'));
        $this->assertSame(123, $token->get('test'));
    }

    public function test_invalid_token()
    {
        $fake = JsonWebToken::parseToken('fake');
        $this->assertfalse($fake->get('valid'));
        $this->assertNull($fake->get('test'));
    }
}
```
