```
<?php declare(strict_types=1);
namespace App\Traits;
use Illuminate\Support\Str;
use Illuminate\Database\Eloquent\Model;
trait UniversallyUnique{
    /**
     * Set Incrementing
     * (Add UUID Key Type to Migration)
     * $table->uuid('id')->primary();
     * @return bool
     */
    public function getIncrementing()
    {
        return false;
    }
    /**
     * Get the Key Type
     * @return string
     */
    public function getKeyType()
    {
        return 'string';
    }

    /**
     * Initialize Identifiable Trait
     * (if you want the key pre-populated for new models)
     * @return void
     */
    public function initializeIdentifiable(){
        //$this->makeUniversallyUnique();
    }

    /**
     * Make Universally Unique
     * @return void
     */
    public function makeUniversallyUnique(){
        if(empty($this->getAttribute($this->getKeyName()))){
            $this->setAttribute($this->getKeyName(), (string) Str::uuid());
        }
    }

    /**
     * Boot the Trait
     * @return void
     */
    public static function bootIdentifiable(){
        static::creating(function(Model $model){
            if(method_exists($model, 'makeUniversallyUnique')){
                $model->makeUniversallyUnique();
            }
        });
    }
}
```