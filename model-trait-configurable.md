## Usage
- Keep in mind it's a collection.  
- You may need to re-assign to itself if you use filter methods etc...

```
$user = request()->user();

if($user->settings->get('subscribed', false)){
    $user->settings->forget('subscribed');
}

if(!$user->settings->has('notify')){
    $user->settings->put('notify', true);
}

$user->save();
```

## Configurable Trait
```
<?php declare(strict_types=1);

namespace App\Traits;
use Illuminate\Support\Collection;
use Illuminate\Database\Eloquent\Model;
trait Configurable{

    /**
     * Get the default settings for the model.
     */
    public static function defaultSettings(): Collection
    {
        return Collection::make([
            "notify" => true, // Send nofications.
            "digest" => true, // Send weekly digest.
        ]);
    }

    /**
     * Boot the Trait
     * @return void
     */
    public static function bootConfigurable(){
        static::creating(function(Model $model){
            $model->setAttribute('settings', self::defaultSettings());
        });
    }
}
```

## Model Cast
```
/**
 * The attributes that should be cast to native types.
 * @var array
 */
protected $casts = [
    'settings' => 'collection',
];
```

## Migration
```
$table->json('settings')->nullable();
```
