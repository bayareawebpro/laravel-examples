## Trait
```
<?php declare(strict_types=1);
namespace App\Models\Traits;
trait ReadableForHumansTrait{
    /**
     * Get Created At
     * @return bool
     */
    public function getCreatedForHumansAttribute()
    {
        return optional($this->created_at)->diffForHumans();
    }
    /**
     * Get Updated At
     * @return bool
     */
    public function getUpdatedForHumansAttribute()
    {
        return optional($this->updated_at)->diffForHumans();
    }
}
```

## Interface
```
<?php declare(strict_types=1);
namespace App\Models\Traits;
interface ReadableForHumans{

    /**
     * Get Created At
     * @return string
     */
    public function getCreatedForHumansAttribute();

    /**
     * Get Updated At
     * @return string
     */
    public function getUpdatedForHumansAttribute();
}

```

