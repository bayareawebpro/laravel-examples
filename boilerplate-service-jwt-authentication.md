# Json Web Token (JWT) Authentication


### Make Encryption Secret
```php
JsonWebToken::generateSecret();
```

### Configure Auth.php
```
'guards' => [
    ...
    'api' => [
        'driver' => 'laravel-jwt', //token
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
use RuntimeException;
use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Crypt;
use Illuminate\Database\Eloquent\Model;

/**
 * JsonWebToken RFC Testing
 * @url https://jwt.io/
 */
class JsonWebToken
{

    protected static Collection $decoded;

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
                return $model::find($token->get('user'));
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
        return static::$decoded ?: (static::$decoded = JsonWebToken::parseToken($token));
    }

    /**
     * Create a new token for a model.
     * @param Model $user
     * @param array $data
     * @param Carbon|null $until
     * @return string
     */
    public static function createForUser(Model $user, ?Carbon $until = null, array $data = []): string
    {
        $until = ($until ?? Carbon::now()->addDays(30));

        $header = static::encodeData([
            "alg" => "HS512",
            "typ" => "JWT",
        ]);
        $payload = static::encodeData(array_merge([
            "user"    => $user->getKey(),
            "expires" => $until->toDateTimeString(),
        ], $data));

        return "{$header}.{$payload}." . static::createSignature($header . $payload);
    }

    /**
     * Parse the token to collection instance.
     * @param string $token
     * @return Collection
     * @throws RuntimeException
     */
    public static function parseToken(?string $token = null): Collection
    {
        $tokenProperties = Collection::make(['valid' => false]);

        list($header, $payload, $signature) = explode('.', $token);

        if (static::verifySignature($header . $payload, $signature)) {
            $tokenProperties = $tokenProperties
                ->merge(static::decodeData($header))
                ->merge(static::decodeData($payload));
            $tokenProperties
                ->put('valid', static::isValidTimestamp($tokenProperties->get('expires')));
        }
        return $tokenProperties;
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
        return hash_hmac(config('auth.jwt.algorithm'), $value, static::getEncryptionKey(), false);
    }

    /**
     * Get Encryption Key
     * @return string
     */
    protected static function getEncryptionKey(): string
    {
        return base64_encode(config('auth.jwt.secret'));
    }

    /**
     * Make Encryption Key
     * @return string
     */
    protected static function generateSecret(): string
    {
        return bin2hex(openssl_random_pseudo_bytes(128));
    }

    /**
     * Verify HMAC Signature
     * @param string $payload
     * @param string $value
     * @return bool
     */
    protected static function verifySignature(string $payload, string $value): bool
    {
        return hash_equals(hash_hmac(config('auth.jwt.algorithm'), $payload, static::getEncryptionKey(), false), $value);
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
        return Crypt::encryptString($token->toBase()->merge($claims)->put('expires', $carbon->toDateTimeString())->toJson());
    }

    /**
     * Is the timestamp claim valid.
     * @param string $timestamp
     * @return bool
     */
    public static function isValidTimestamp(string $timestamp): bool
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
