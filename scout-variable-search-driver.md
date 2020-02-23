# Scout Variable Search Driver

Configure the driver per-model:

> Source: https://dev.to/summitech/laravel-how-to-configure-multiple-search-drivers-65

```php
<?php

use Laravel\Scout\EngineManager;

public function searchableUsing()
{
   return app(EngineManager::class)->engine('algolia');
}
```