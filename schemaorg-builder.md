```
$schema = SchemaBuilder::make(config('my.schema.org.config'));
$schema->setAttribute('nested.key', 'Other Name');
$schema->hasAttribute('nested.key');
$schema->removeAttribute('parentKey');
$schema->toJson();
$schema->toArray();
$schema->toHtml();
$schema->render();
```

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