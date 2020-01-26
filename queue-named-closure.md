## Call Named Queued Closure

Allow Naming Queued Closures.

```php
<?php

namespace App\Utilities;
use Illuminate\Queue\CallQueuedClosure;

class CallNamedQueuedClosure extends CallQueuedClosure
{
    protected $displayName = null;

    public function setDisplayName(string $value)
    {
        return $this->displayName = $value;
    }

    public function displayName()
    {
        $name = $this->displayName ?? parent::displayName();
        return "(Closure) {$name}";
    }
}
```