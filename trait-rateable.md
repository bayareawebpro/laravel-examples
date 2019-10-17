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
    $model->rate($request->user(), $request->validationData());
    $model->isRated;
    $model->rating;
}
```

## Request

```
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class RatingRequest extends FormRequest
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
        return  [
            'rating' => 'required|numeric|min:1,max:5'
        ];
    }
}
```

## Trait

```
<?php declare(strict_types=1);

namespace App\Models\Traits;
use Illuminate\Database\Eloquent\Builder;
use App\Models\Rating;
use App\Models\User;

trait RateableTrait{

    /**
     * Ratings Relation
     * @return \Illuminate\Database\Eloquent\Relations\MorphToMany
     */
    public function ratings(){
        return $this->morphMany(Rating::class, 'model');
    }

    /**
     * Get Is Rated Attribute
     * @return bool
     */
    public function getIsRatedAttribute(): bool
    {
        if($user = request()->user()){
            return (bool) $this->ratedBy($user)->exists();
        }
        return false;
    }

    /**
     * Get Rated Attribute
     * @return int
     */
    public function getRatingAttribute(): int
    {
        return (int) round($this->ratings()->average('rating'));
    }

    /**
     * Get Articles for User
     * @param User $user
     * @param array $data
     * @return self
     * @throws \Throwable
     */
    public function rate(User $user, array $data)
    {
        if ($this->isRated) {
            $this
                ->ratings()
                ->where('user_id', $user->id)
                ->update($data);
        } else {
            $model = new Rating($data);
            $model->user()->associate($user);
            $this->ratings()->save($model);
        }
        return $this;
    }

    /**
     * Rated By Ownable
     * @param Builder $query
     * @param User $user
     * @return Builder
     */
    public function scopeRatedBy(Builder $query, User $user): Builder
    {
        return $query->whereHas('ratings', function(Builder $query) use ($user){
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
interface Rateable{

    /**
     * Ratings Relation
     * @return MorphToMany
     */
    public function ratings(): MorphToMany;

    /**
     * Get Is Rated Attribute
     * @return bool
     */
    public function getIsRatedAttribute(): bool;

    /**
     * Get Rated Attribute
     * @return int
     */
    public function getRatingAttribute(): int;

    /**
     * Get Articles for User
     * @param User $user
     * @param array $data
     * @return self
     * @throws \Throwable
     */
    public function rate(User $user, array $data): self;

    /**
     * Rated By Ownable
     * @param Builder $query
     * @param User $user
     * @return Builder
     */
    public function scopeRatedBy(Builder $query, User $user): Builder;
}
```

## Migration

```
<?php
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateRatingsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('ratings', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->uuid('user_id')->index();
            $table->uuid('model_id');
            $table->string('model_type');
            $table->index(['model_id', 'model_type']);
            $table->tinyInteger('rating')->default(0)->index();
            $table->timestamps();
        });
        Schema::table('ratings', function (Blueprint $table) {
            $table
                ->foreign('user_id', 'ratings_user_foreign')
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
        Schema::dropIfExists('ratings');
    }
}
```


## Model

```
<?php

namespace App\Models;
use Illuminate\Database\Eloquent\Model;

class Rating extends Model{

    /**
     * Fillable Attributes
     * @var array
     */
    protected $fillable = [
        "rating",
    ];

    /**
     * MorphTo Rated Model
     * @return \Illuminate\Database\Eloquent\Relations\MorphTo
     */
    public function rated()
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
 * Ratings
 * @return \Illuminate\Database\Eloquent\Relations\hasMany
 */
public function ratings()
{
    return $this->hasMany(Rating::class);
}
```