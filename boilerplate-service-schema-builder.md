## Schema Builder
- Designed for any type of schema.
- Example usage with Schema.org for SEO.

## Register Global Schema Collection.
```
use App\Schema\SchemaBuilder;

$this->app->singleton('schemaOrg', function(){
    return new Collection([
        SchemaBuilder::make(config('seo.schema.website'))
    ]);
});
```

### Push Schema to Collection
```
use App\Schema\SchemaBuilder;

$schema = SchemaBuilder::make(config('seo.schema.article'), [
    'title' => 'My Article'
]);

app('schemaOrg')->push($schema);
```

## Render Helper
```
/**
 * Embed Schema
 * @return string
 * @usage {!! schemaOrg() !!}
 */
function schemaOrg()
{
    try{
        return app('schemaOrg')->map(function (\App\Schema\SchemaBuilder $schema) {
            return $schema->toHtml();
        })->implode("\n");
    }catch (\Exception $e){
        logger()->critical($e->getMessage(), $e->getTrace());
    }
}
```

## Schema Builder Usage
```
$schema = SchemaBuilder::make(config('my.schema.org.config'), $mergeAttributes = []);
$schema->setAttribute('nested.key', 'Other Name');
$schema->hasAttribute('nested.key');
$schema->removeAttribute('parentKey');
$schema->toJson();
$schema->toArray();
$schema->toHtml();
$schema->render();
```

```
$reviews = Repository::make()
    ->testimonials(['great','amazing','recommended'],4)
    ->map(function($entry){
       return SchemaBuilder::make(
               config('cms.schema.organization'), 
               $entry->toArray()
           )->toArray();
    })->values();

SchemaBuilder::make(config('cms.schema.organization'))->setAttributes([
    "aggregateRating" => config('cms.schema.aggregateRating'),
    "reviews" => $reviews,
]);
```

## Schema Builder Class
```
<?php
declare(strict_types=1);

namespace App\Schema;
use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Contracts\Support\Htmlable;
use Illuminate\Contracts\Support\Jsonable;
use Illuminate\Contracts\Support\Renderable;

class SchemaBuilder implements Jsonable, Htmlable, Renderable, Arrayable
{
    /** @var array $schema */
    protected $schema;

    /**
     * Make Schema
     * @param array $schema
     * @param array $attributes
     * @return self
     */
    public static function make($schema = array(), $attributes = array()): SchemaBuilder
    {
        return new self($schema, $attributes);
    }

    /**
     * SchemaBuilder constructor.
     * @param array $schema
     * @param array $attributes
     */
    protected function __construct(array $schema, array $attributes)
    {
        $this->schema = $schema;
        $this->setAttributes($attributes);
    }

    /**
     * Set Many Attributes.
     * @param array $attributes
     * @return self
     */
    public function setAttributes($attributes): SchemaBuilder
    {
        collect($attributes)->each(function($value, $key){
            $this->setAttribute($key, $value);
        });
        return $this;
    }

    /**
     * Set a Single Attribute.
     * @param string $key
     * @param mixed $value
     * @return self
     */
    public function setAttribute($key, $value): SchemaBuilder
    {
        if(!empty($value)){
            data_set($this->schema, $key, $value);
        }
        return $this;
    }

    /**
     * Remove a Single Attribute.
     * @param string $key
     * @return self
     */
    public function removeAttribute($key): SchemaBuilder
    {
        unset($this->schema[$key]);
        return $this;
    }

    /**
     * Remove a Single Attribute.
     * @param string $key
     * @return bool
     */
    public function hasAttribute($key): bool
    {
        return !empty($this->getAttribute($key));
    }

    /**
     * Get a Single (nested) Attribute.
     * @param string $key
     * @param mixed $fallback
     * @return mixed
     */
    public function getAttribute($key, $fallback = null)
    {
        return data_get($this->schema, $key, $fallback);
    }

    /**
     * Get the evaluated JSON string of the object.
     * @param int $options
     * @return string
     */
    public function toJson($options = 0): string
    {
        return json_encode(array_merge(array("@context" => "https://schema.org"), $this->schema), $options);
    }

    /**
     * Get the evaluated array of the object.
     * @return array
     */
    public function toArray(): array
    {
        return (array) $this->schema;
    }

    /**
     * Render to Html
     * @return string
     */
    public function toHtml(): string
    {
        $output = '<script type="application/ld+json" class="schema-org">';
        $output .= $this->toJson();
        $output .= '</script>';
        return $output;
    }

    /**
     * Render the evaluated script tag.
     * @return string
     */
    public function render(): string
    {
        return $this->toHtml();
    }
}

```