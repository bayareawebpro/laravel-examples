## Selectable JSON Resource

```php
SelectionResource::withLabel('full_name');
$formatted = SelectionResource::make(Model::all());
```

```php
<?php
namespace App\Http\Resources;
use Illuminate\Http\Resources\Json\JsonResource;

class SelectionResource extends JsonResource{

    public static $label = 'name';

    public static function withLabel($attribute){
        static::$label = $attribute;
    }

    public function toArray($request)
    {
        return [
            'value' => $this->id,
            'label' => $this->{static::$label},
        ];
    }
}
```