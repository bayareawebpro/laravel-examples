## ArrayIsCorrectSizeAndOrder Rule

> Author: @GDanielRG
> Source: https://gist.github.com/GDanielRG/580cc7c402475e1e74c9198bf151aafd

```php
[
    'variants.*.options' => [new ArrayIsCorrectSizeAndOrder(array_column($this->options ?: [], 'variants'))]
];
```

```php
<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class ArrayIsCorrectSizeAndOrder implements Rule
{
    protected $reference;

    /**
     * Create a new rule instance.
     * @return void
     */
    public function __construct(array $reference)
    {
        $this->reference = $reference;
    }

    /**
     * Determine if the validation rule passes.
     * @param  string  $attribute
     * @param  mixed  $value
     * @return bool
     */
    public function passes($attribute, $value)
    {
        if(sizeof($value) != sizeof($this->reference)){
            return false;
        }

        for ($i=0; $i < sizeof($value); $i++) { 
            if(!in_array($value[$i], $this->reference[$i])){
                return false;
            }
        }
        return true;
    }

    /**
     * Get the validation error message.
     * @return string
     */
    public function message()
    {
        return 'The :attribute field does not matches the specified size or order.';
    }
}
```