```
<?php declare(strict_types=1);

namespace App\Traits;

trait DetectsEdits{

    /**
     * Get Was Edited Conditional
     * @return mixed
     */
    public function getWasEditedAttribute(): bool
    {
        return optional($this->created_at)->notEqualTo($this->updated_at) || false;
    }
}

```