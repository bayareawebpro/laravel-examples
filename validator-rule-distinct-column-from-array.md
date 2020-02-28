## DistinctColumnFromArray Rule

> Author: @GDanielRG


```php
[
    'variants' => [new DistinctColumnFromArray('options')]
];
```

```php

<?php

namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class DistinctColumnFromArray implements Rule
{
    protected $_columnName;

    /**
     * Create a new rule instance.
     *
     * @return void
     */
    public function __construct($columnName)
    {
        $this->_columnName = $columnName;
    }

    /**
     * Determine if the validation rule passes.
     *
     * @param  string  $attribute
     * @param  mixed  $value
     * @return bool
     */
    public function passes($attribute, $value)
    {
        return $this->array_has_duplicates(array_column($value, $this->_columnName));
    }

    /**
     * Get the validation error message.
     *
     * @return string
     */
    public function message()
    {
        return 'The :attribute.' . $this->_columnName . ' field must be unique for all entries.';
    }

    /**
     * Determines if the input array has at least one duplicate
     */
    protected function array_has_duplicates(array $input_array) {
        $startingPoint = 1;
        for ($i=0; $i < sizeof($input_array) - 1; $i++) { 
            for ($j=$startingPoint; $j < sizeof($input_array); $j++) { 
                if($input_array[$i] == $input_array[$j]){
                    return false;
                }
            }
            $startingPoint++;
        }
        return true;
    }
}
```