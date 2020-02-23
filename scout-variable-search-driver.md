# Scout Variable Search Driver

Configure the driver per-model:

```php
<?php

use Laravel\Scout\EngineManager;

public function searchableUsing()
{
   return app(EngineManager::class)->engine('algolia');
}
```