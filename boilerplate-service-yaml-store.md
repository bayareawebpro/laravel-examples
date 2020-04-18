# YAML Service

```json
"symfony/yaml": "^5.0"
```

```php
Yaml::make('test.yaml', 'local')
    ->set('apps.test', [
        'php' => 7.4,
        'path' => '/home/me/app',
        'groups' => [
            'me', 'www-data/www-data',
        ],
    ])
->save();
```


```php
$yaml = Yaml::make('test.yaml', 'local');

$yaml->set('groups', [
    'me/me', 
    'wheel/wheel',
    'www-data/www-data',
]);

$groups = $yaml->collect('groups');

$yaml->set('groups',
    $groups->reject(function($group){
        return $group === 'me/me';
    })
);

$yaml->save();
```


## Service

```php
<?php declare(strict_types=1);

namespace App;

use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Storage;
use Symfony\Component\Yaml\Yaml as YamlParser;

class Yaml
{
    /**
     * Yaml File Data
     * @var array
     */
    protected $data = [];

    /**
     * File Path
     * @var string|null
     */
    protected $filepath = null;

    /**
     * Storage Disk
     * @var string
     */
    protected $disk = 'local';

    /**
     * Yaml constructor.
     * @param string $filepath
     * @param string $disk
     */
    public function __construct(string $filepath, string $disk)
    {
        $this->filepath = $filepath;
        $this->disk = $disk;
        $this->data = [];
        $this->load();
    }

    /**
     * Static Make Method
     * @param string $filepath
     * @param string $disk
     * @return static
     */
    public static function make(string $filepath, string $disk = 'local'): self
    {
        return new static($filepath, $disk);
    }

    /**
     * Load Yaml File
     * @return $this
     */
    public function load(): self
    {
        if(Storage::disk($this->disk)->exists($this->filepath)){
            $data = YamlParser::parse(
                Storage::disk($this->disk)->get($this->filepath),
                YamlParser::PARSE_OBJECT_FOR_MAP
            );
            $this->data = is_array($data) ? $data : [];
        }
        return $this;
    }

    /**
     * Get Attribute
     * @param string $key
     * @param null $fallback
     * @return mixed
     */
    public function get(string $key, $fallback = null)
    {
        return data_get($this->data, $key, $fallback);
    }

    /**
     * Collect Attribute
     * @param string $key
     * @return Collection
     */
    public function collect($key = null): Collection
    {
        return Collection::make(isset($key) ? $this->get($key, []) : $this->data);
    }

    /**
     * Set Attribute
     * @param string $key
     * @param $value
     * @return $this
     */
    public function set(string $key, $value): self
    {
        data_set($this->data, $key,
            method_exists($value, 'toArray') ? $value->toArray() : $value
        );
        return $this;
    }

    /**
     * Save Yaml File to Disk.
     * @return $this
     */
    public function save(): self
    {
        Storage::disk($this->disk)->put($this->filepath,
            YamlParser::dump($this->data,PHP_INT_MAX, 4,YamlParser::DUMP_OBJECT_AS_MAP)
        );
        return $this;
    }
}
```
