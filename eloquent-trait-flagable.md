## Usage

```
/**
 * Flag the Resource
 * @param FlagRequest $request
 * @param Model $model
 * @return \Illuminate\Http\Response
 * @throws \Throwable
 */
public function flag(FlagRequest $request, Model $model)
{
    $model->flag($request->user(), $request->all());
    $model->isFlagged;
}
```

## Request
```
<?php declare(strict_types=1);

namespace App\Http\Requests;
use Illuminate\Foundation\Http\FormRequest;

class FlagRequest extends FormRequest
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
     * Get the validation rules that apply to the request.
     * @return array
     */
    public function rules()
    {
        return [
            'reason' => 'required|string|max:1000'
        ];
    }
}
```

## Trait
```
<?php declare(strict_types=1);

namespace App\Models\Traits;
use App\Models\Favorite;
use App\Models\Flagged;
use App\Models\User;
use PhpParser\Builder;

trait FlaggableTrait{

    /**
     * Flagged Relation
     * @return \Illuminate\Database\Eloquent\Relations\MorphToMany
     */
    public function flagged(){
        return $this->morphMany(Flagged::class, 'model');
    }

    /**
     * Get Is Flagged Attribute
     * @return bool
     */
    public function getIsFlaggedAttribute(): bool
    {
        if($user = request()->user()){
            return (bool) $this->flaggedBy($user)->exists();
        }
        return false;
    }

    /**
     * Flag Questionable Model
     * @param User $user
     * @param array $data
     * @return self
     */
    public function flag(User $user, array $data):self
    {
        if($this->isFlagged){
            $this->flagged()->where('user_id', $user->id)->delete();
        }else{
            $model = new Flagged($data);
            $model->user()->associate($user);
            $this->flagged()->save($model);
        }
        return $this;
    }

    /**
     * Flagged By Ownable
     * @param User $user
     * @return Builder
     */
    public function scopeFlaggedBy(Builder $query, User $user): Builder
    {
        return $this->whereHas('flagged', function(Builder $query) use ($user){
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
use Illuminate\Database\Eloquent\Relations\MorphToMany

interface Flaggable{

    /**
     * Flagged Relation
     * @return MorphToMany
     */
    public function flagged(): MorphToMany;

    /**
     * Get Is Bookmarked Attribute
     * @return bool
     */
    public function getIsFlaggedAttribute(): bool;

    /**
     * Flag Questionable Model
     * @param User $user
     * @param array $data
     * @return self
     */
    public function flag(User $user, array $data):self;

    /**
     * Flagged By Ownable
     * @param Builder $query
     * @param User $user
     * @return Builder
     */
    public function scopeFlaggedBy(Builder $query, User $user): Builder;
}
```

## Migration
```
<?php
use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateFlaggedTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('flagged', function (Blueprint $table) {
            $table->uuid('id')->primary();
            $table->uuid('user_id')->index();
            $table->uuid('model_id');
            $table->string('model_type');
            $table->index(['model_id', 'model_type']);
            $table->text('reason')->nullable();
            $table->timestamps();
        });
        Schema::table('flagged', function (Blueprint $table) {
            $table
                ->foreign('user_id', 'flagged_user_foreign')
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
        Schema::dropIfExists('flagged');
    }
}
```

## Model

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Flagged extends Model{

    protected $table = 'flagged';

    /**
     * Fillable Attributes
     * @var array
     */
    protected $fillable = [
        "reason",
    ];

    /**
     * Questionable Model
     * @return \Illuminate\Database\Eloquent\Relations\MorphTo
     */
    public function questionable()
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
 * Flagged
 * @return \Illuminate\Database\Eloquent\Relations\hasMany
 */
public function flagged()
{
    return $this->hasMany(Flagged::class);
}
```