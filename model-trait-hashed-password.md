## HashedPassword Trait

- IMO, there should be no possibility to set a password attribute that is not hashed.  
- Remove the hash call in your registration controller in favor of allowing the model to do it automatically.
- This may not be compatible with Laravel Nova's password reset functionality.

```
<?php declare(strict_types=1);
 
namespace App\Traits;
use Illuminate\Support\Facades\Hash;
 
trait HashedPassword{
    /**
     * Set the password attribute.
     * @param $value
     */
    public function setPasswordAttribute($value)
    {
        if (isset($value)) {
            $this->attributes['password'] = Hash::make($value);
        }
    }
}
 
 ```