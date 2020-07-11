# Json Web Token (JWT) Authentication


### Make Encryption Secret
```php
JsonWebToken::generateSecret();
```

### Add Secret to Environment File
```php
JWT_SECRET=xxx
```

### Configure Auth.php
```
'guards' => [
    ...
    'api' => [
        'driver' => 'laravel-jwt', 
        'provider' => 'users',
        'hash' => false,
    ],
],
'jwt' => [
    'secret' => env('JWT_SECRET'),
    'algorithm' => 'sha256',
],
```

### Register in Auth Service Provider.

```php
JsonWebToken::register(User::class, 'token');
```

### Create New Token
```php
$token = JsonWebToken::createForUser(User::first(), now()->addHours(3), [
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

### Extend Token Lifetime & Claims
```php
$newToken = JsonWebToken::extendToken(request()->jwt(), now()->addHours(3), ['key' => true]);
```

### Service Class
```php
<?php declare(strict_types=1);

namespace App\Services;

use stdClass;
use Carbon\Carbon;
use Illuminate\Support\Str;
use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\App;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Config;
use Illuminate\Database\Eloquent\Model;

/**
 * JsonWebToken RFC Testing
 * @url https://jwt.io/
 */
class JsonWebToken
{
    /**
     * Register The Model & Macros
     * @param string $model
     * @param string $keyName
     */
    public static function register(string $model, string $keyName = 'token'): void
    {
        Auth::viaRequest('laravel-jwt', function (Request $request) use ($model) {
            $token = $request->jwt();
            if ($token->get('valid')) {
                return $model::query()->find($token->get('user'));
            }
        });

        Request::macro('jwt', function (?string $key = null) use ($keyName) {
            $token = JsonWebToken::singletonInstance($this->get($keyName));
            if ($key) {
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
        $app = App::getFacadeRoot();
        return $app->bound('jwt-decoded')
            ? $app->get('jwt-decoded')
            : $app->instance('jwt-decoded', JsonWebToken::parseToken($token));
    }

    /**
     * Create a new token for a model.
     * @param Model $user
     * @param array $data
     * @param Carbon|null $expires
     * @return string
     */
    public static function createForUser(Model $user, ?Carbon $expires = null, array $data = []): string
    {
        $headers = static::getHeaders();
        $payload = static::encodeData(array_merge([
            "expires" => ($expires ?? Carbon::now()->addDays(30))->toDateTimeString(),
            "user"    => $user->getKey(),
        ], $data));

        return "{$headers}.{$payload}." . static::createSignature("{$headers}.{$payload}");
    }

    /**
     * Parse the token to collection instance.
     * @param string $token
     * @return Collection
     */
    public static function parseToken(?string $token = null): Collection
    {
        if (Str::substrCount($token, '.') === 2) {
            list($headers, $payload, $signature) = explode('.', $token);
            if (static::verifySignature("{$headers}.{$payload}", $signature)) {
                $tokenProperties = Collection::make(static::decodeData($payload));
                $tokenProperties->put('valid', static::isValidTimestamp($tokenProperties->get('expires')));
                return $tokenProperties;
            }
        }
        return Collection::make(['valid' => false]);
    }

    /**
     * Decode Data
     * @param string $value
     * @return stdClass
     */
    protected static function decodeData(string $value): stdClass
    {
        return json_decode(base64_decode(strtr($value, '-_', '+/')));
    }

    /**
     * Encode Data
     * @param array $value
     * @return string
     */
    protected static function encodeData(array $value): string
    {
        return rtrim(strtr(base64_encode(json_encode($value)), '+/', '-_'), '=');
    }

    /**
     * Create HMAC Signature
     * @param string $value
     * @return string
     */
    protected static function createSignature(string $value): string
    {
        return hash_hmac(static::getAlgorithm(), $value, static::getEncryptionKey(), false);
    }

    /**
     * Get Encryption Key
     * @return string
     */
    protected static function getEncryptionKey(): string
    {
        return base64_encode(Config::get('auth.jwt.secret'));
    }

    /**
     * Get Encryption Key
     * @return string
     */
    protected static function getAlgorithm(): string
    {
        return Config::get('auth.jwt.algorithm');
    }

    /**
     * Make Encryption Key
     * @param int $length
     * @return string
     */
    public static function generateSecret(int $length = 64): string
    {
        return Str::random($length);
    }

    /**
     * Verify HMAC Signature
     * @param string $payload
     * @param string $value
     * @return bool
     */
    protected static function verifySignature(string $payload, string $value): bool
    {
        return hash_equals($value, hash_hmac(static::getAlgorithm(), $payload, static::getEncryptionKey(), false));
    }

    /**
     * Extend Expires Claim
     * @param Collection $token
     * @param Carbon $carbon
     * @param array $claims
     * @return string
     */
    public static function extendToken(Collection $token, Carbon $carbon, array $claims = []): string
    {
        $headers = static::getHeaders();
        $payload = static::encodeData(
            $token
                ->toBase()
                ->except(['valid', 'alg', 'typ'])
                ->put('expires', $carbon->toDateTimeString())
                ->merge($claims)
                ->toArray()
        );
        return "{$headers}.{$payload}." . static::createSignature("{$headers}.{$payload}");
    }

    /**
     * Get the token headers.
     * @return string
     */
    protected static function getHeaders(): string
    {
        return static::encodeData([
            "alg" => "HS512",
            "typ" => "JWT",
        ]);
    }

    /**
     * Is the timestamp claim valid.
     * @param string|null $timestamp
     * @return bool
     */
    public static function isValidTimestamp(?string $timestamp = null): bool
    {
        return Carbon::parse($timestamp)->greaterThanOrEqualTo(Carbon::now());
    }
}
```

### Feature Tests

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
        $token = JsonWebToken::createForUser($user);

        $this
            ->json('GET',"/api/user?token={$token}")
            ->assertStatus(200)
            ->assertJson($user->toArray())
        ;
    }

    public function test_valid_token()
    {
        $token = JsonWebToken::parseToken(JsonWebToken::createForUser(
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

    public function test_extend_token()
    {
        $user = factory(User::class)->create();
        $token = JsonWebToken::createForUser($user, now()->addHours(1));
        $token = JsonWebToken::parseToken($token);
        $extended = now()->addHours(2);
        $newToken = JsonWebToken::extendToken($token, $extended);
        $newToken = JsonWebToken::parseToken($newToken);
        $this->assertNotSame($token->get('expires'), $newToken->get('expires'));
        $this->assertSame($newToken->get('expires'), $extended->toDateTimeString());
    }
}

```
