## Usage

```
/**
 * Favorite the Resource
 * @param FavoriteRequest $request
 * @param Model $model
 * @return \Illuminate\Http\Response
 * @throws \Throwable
 */
public function favor(FavoriteRequest $request, Model $model)
{
    $model->favor($request->user());
    $model->isFavored;
}
```

## Request

```
<?php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class FavoriteRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     * @return bool
     */
    public function authorize()
    {
        return !is_null($this->user());
    }

    /**
     * Get the validated data from the request.
     * @return array
     */
    public function validationData()
    {
        return $this->all();
    }

    /**
     * Get the validation rules that apply to the request.
     * @return array
     */
    public function rules()
    {
        return  [];
    }
}

```

## Trait

```
<?php declare(strict_types=1);

namespace App\Models\Traits;

use App\Models\User;
use App\Models\Favorite;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Relations\MorphToMany;

trait FavorableTrait{

    /**
     * Favorites Relation
     * @return MorphToMany
     */
    public function favorites(): MorphToMany
    {
        return $this->morphMany(Favorite::class, 'model');
    }

    /**
     * Get Is Favored Attribute
     * @return bool
     */
    public function getIsFavoredAttribute(): bool
    {
        return (bool) $this
            ->favorites()
            ->where('favorites.user_id', optional(request()->user())->id)
            ->exists();
    }

    /**
     * Add Favorite
     * @param User $user
     * @return self
     */
    public function favor(User $user):self
    {
        if ($this->isFavored) {
            $this->favoredBy(request()->user())->delete();
        } else {
            $model = new Favorite();
            $model->user()->associate($user);
            $this->favorites()->save($model);
        }
        return $this;
    }

    /**
     * Favored By Ownable
     * @param Builder $query
     * @param User $user
     * @return Builder
     */
    public function scopeFavoredBy(Builder $query, User $user): Builder
    {
        return $query->whereHas('favorites', function(Builder $query) use ($user){
            $query->where('user_id', $user->id);
        });
    }
}

```

## Interface

```
<?php declare(strict_types=1);

namespace App\Models\Traits;

use App\Models\User;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Relations\MorphToMany;

interface Favorable{

    /**
     * Favorites Relation
     * @return MorphToMany
     */
    public function favorites(): MorphToMany;

    /**
     * Get Is Bookmarked Attribute
     * @return bool
     */
    public function getIsFavoredAttribute(): bool;

    /**
     * Add Favorite
     * @param User $user
     * @return self
     */
    public function favor(User $user): self;

    /**
     * Favored By Ownable
     * @param Builder $query
     * @param User $user
     * @return Builder
     */
    public function scopeFavoredBy(Builder $query, User $user): Builder;
}
```

## Migration

```
<?php
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateFavoritesTable extends Migration
{
    /**
     * Run the migrations.
     * @return void
     */
    public function up()
    {
        Schema::create('favorites', function (Blueprint $table) {
            $table->uuid("id")->primary();
            $table->uuid("user_id")->index();
            $table->uuid("model_id");
            $table->string("model_type");
            $table->index(["model_type", "model_id"]);
            $table->timestamps();
        });
        Schema::table('favorites', function (Blueprint $table) {
            $table
                ->foreign('user_id', 'favorites_user_foreign')
                ->references('id')
                ->on('users')
                ->onDelete('CASCADE')
                ->onUpdate('NO ACTION');
        });
    }

    /**
     * Reverse the migrations.
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('favorites');
    }
}
```
## Model
```
<?php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Favorite extends Model{
   
    /**
     * Fillable Attributes
     * @var array
     */
    protected $fillable = [];

    /**
     * Favored Model
     * @return \Illuminate\Database\Eloquent\Relations\MorphTo
     */
    public function favored()
    {
        return $this->morphTo('model');
    }

    /**
     * BelongsTo User
     * @return \Illuminate\Database\Eloquent\Relations\BelongsTo
     */
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

## User Model Relation
```
/**
 * Favorites
 * @return \Illuminate\Database\Eloquent\Relations\hasMany
 */
public function favorites()
{
    return $this->hasMany(Favorite::class);
}
```