
## Filterable Request Trait

Will call a possible filter method for every property.

```
<?php declare(strict_types=1);

namespace App\Http\Requests\Traits;
use Illuminate\Support\Collection;

trait Filterable{
    /**
     * Get the validated data from the request.
     * @return array
     */
    public function validated()
    {
        return Collection::make(parent::validated())->map(function ($value, $key) {
            $filter = "{$key}Filter";
            if (method_exists($this, $filter)) {
                return $this->{$filter}($value);
            }
            return $value;
        })->all();
    }
}

```


```
<?php

namespace App\Http\Requests;

use App\Post;
use Illuminate\Support\Collection;
use App\Http\Requests\Traits\Filterable;
use Illuminate\Foundation\Http\FormRequest;

class PostRequest extends FormRequest
{
    use Filterable;

    /**
     * Determine if the user is authorized to make this request.
     * @return bool
     */
    public function authorize()
    {
        return true;
    }

    /**
     * Get the validation rules that apply to the request.
     * @return array
     */
    public function rules()
    {
        return [
            ...
        ];
    }

    /**
     * Content Attribute Filter.
     * @param $value
     * @return string
     */
    protected function contentFilter($value)
    {
        return strip_tags((string) $value, Collection::make([
            'table', 'thead', 'tbody', 'tfoot', 'th', 'tr', 'td',
            'a', 'p', 'ul', 'ol', 'li', 'em', 'strong', 'blockquote',
            'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
            'img', 'iframe', 'figure',
        ])->map(function ($tag) {
            return "<$tag>";
        })->join(''));
    }
}

```