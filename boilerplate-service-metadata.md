```php
 MetaData::make()
    ->title("Create Review")
    ->description("Submit a Review.")
    ->robots('index,nofollow');
```

```php
<?php declare(strict_types=1);

namespace Services;

use Illuminate\Config\Repository;

class MetaData
{
    protected $config;

    public static function make(): MetaData
    {
        return app(static::class);
    }

    public function __construct(Repository $config)
    {
        $this->config = $config;
    }

    public function title(string $value)
    {

        $this->config->set('cms.seo.meta_title', $value);
        return $this;
    }

    public function description(string $value)
    {

        $this->config->set('cms.seo.meta_description', $value);
        return $this;
    }

    public function robots(string $value)
    {

        $this->config->set('cms.seo.meta_robots', $value);
        return $this;
    }
}
```