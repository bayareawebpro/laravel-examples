### Laravel AirLock Token CRUD

```php
Route::group(['middleware' => 'auth:airlock'], function(){
    Route::resource('tokens','TokenController');
});
```

```php

<?php
namespace App\Http\Controllers;

use App\Models\ApiToken;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Validation\ValidationException;
use Illuminate\Auth\Access\AuthorizationException;

class TokenController extends Controller
{
    public function __construct()
    {
        $this->authorizeResource(ApiToken::class, 'token');
    }

    /**
     * Display a listing of the resource.
     * @param Request $request
     * @return SearchableResource
     */
    public function index(Request $request)
    {
        return response([
            'resource' => $request->user()->tokens()->with('tokenable')->paginate(8)
        ]);
    }

    /**
     * Show the form for creating a new resource.
     * @return Response
     */
    public function create(): Response
    {
        return response([
            'entity' => [
                'name' => '',
                'abilities' => [],
            ]
        ]);
    }

    /**
     * Display the specified resource.
     * @param ApiToken $token
     * @return Response
     */
    public function show(ApiToken $token): Response
    {
        return response([
            'entity' => $token
        ]);
    }

    /**
     * Display the specified resource.
     * @param ApiToken $token
     * @return Response
     */
    public function edit(ApiToken $token): Response
    {
        return response([
            'entity' => $token
        ]);
    }

    /**
     * Store a newly created resource in storage.
     * @param Request $request
     * @return Response
     * @throws AuthorizationException|ValidationException
     */
    public function store(Request $request): Response
    {
        $this->authorize('create', [ApiToken::class]);

        $request->validate(ApiToken::validationRules());

        $newAccessToken = $request->user()->createToken(
            $request->get('name'),
            $request->get('abilities')
        );

        return response([
            'message' => 'Entity Created',
            'entity' => array_merge($newAccessToken->accessToken->toArray(), [
                'token' => $newAccessToken->plainTextToken
            ]),
        ]);
    }

    /**
     * Update the specified resource in storage.
     * @param Request $request
     * @param ApiToken $token
     * @return Response
     */
    public function update(Request $request, ApiToken $token): Response
    {
        $request->validate(ApiToken::validationRules($token));

        return response([
            'message' => 'Entity Updated',
            'entity' => tap($token)->update($request->only(['name', 'abilities']))
        ]);
    }

    /**
     * Remove the specified resource from storage.
     * @param ApiToken $token
     * @return Response
     * @throws \Exception
     */
    public function destroy(ApiToken $token): Response
    {
        $token->delete();

        return response([
            'message' => 'Entity Destroyed',
            'entity' => $token->only('id')
        ]);
    }
}
```

### Token Model
```php
<?php
namespace App\Models;

use App\Services\AppPermissions;
use Illuminate\Validation\Rule;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Cache;
use Laravel\Airlock\PersonalAccessToken;

class ApiToken extends PersonalAccessToken
{
    public $table = 'personal_access_tokens';

    /**
     * The validation rules for an entity.
     * @return array
     */
    public static function validationRules(): array
    {
        return [
            'name'        => 'required|string|max:255',
            'abilities'   => 'required|array|min:1',
            'abilities.*' => ['string', Rule::in([ 
            // ...AllowedGrants
            ])],
        ];
    }

    public static function abilitiesInUse(): Collection
    {
        return Cache::remember('airlock:abilities', 120, fn()=>static::query()
            ->pluck('abilities')
            ->flatten(1)
            ->unique()
            ->values());
    }

    public static function boot()
    {
        parent::boot();
        static::saved(fn()=>Cache::forget('airlock:abilities'));
        static::deleted(fn()=>Cache::forget('airlock:abilities'));
    }
}
```

### Policy
```php
<?php
namespace App\Policies;

use App\Models\User;
use App\Models\ApiToken;
use Illuminate\Auth\Access\HandlesAuthorization;

class ApiTokenPolicy
{
    use HandlesAuthorization;

    /**
     * Determine whether the user can view any tokens.
     * @param User $user
     * @return bool
     */
    public function viewAny(User $user): bool
    {
        return $user->isRole('admin')
            || $user->tokenCan('tokens:viewAny');
    }

    /**
     * Determine whether the user can view the token.
     * @param User $user
     * @param ApiToken $token
     * @return bool
     */
    public function view(User $user, ApiToken $token): bool
    {
        return $user->isRole('admin')
            || ($user->is($token->tokenable) && $user->tokenCan('tokens:view'));
    }

    /**
     * Determine whether the user can create tokens.
     * @param User $user
     * @return bool
     */
    public function create(User $user): bool
    {
        return $user->isRole('admin')
            || $user->tokenCan('tokens:create');
    }

    /**
     * Determine whether the user can update the token.
     * @param User $user
     * @param ApiToken $token
     * @return bool
     */
    public function update(User $user, ApiToken $token): bool
    {
        return $user->isRole('admin')
            || ($user->is($token->tokenable) && $user->tokenCan('tokens:update'));
    }

    /**
     * Determine whether the user can delete the token.
     * @param User $user
     * @param ApiToken $token
     * @return bool
     */
    public function delete(User $user, ApiToken $token): bool
    {
        return $user->isRole('admin')
            || ($user->is($token->tokenable) && $user->tokenCan('tokens:delete'));
    }

    /**
     * Determine whether the user can restore the token.
     * @param User $user
     * @param ApiToken $token
     * @return bool
     */
    public function restore(User $user, ApiToken $token): bool
    {
        return $user->isRole('admin')
            || ($user->is($token->tokenable) && $user->tokenCan('tokens:restore'));
    }

    /**
     * Determine whether the user can permanently delete the token.
     * @param User $user
     * @param ApiToken $token
     * @return bool
     */
    public function forceDelete(User $user, ApiToken $token): bool
    {
        return $user->isRole('admin')
            || ($user->is($token->tokenable) && $user->tokenCan('tokens:forceDelete'));
    }
}
```