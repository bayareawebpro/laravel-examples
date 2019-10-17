## Environment
```
SCOUT_DRIVER=tntsearch
TNTSEARCH_FUZZINESS=true
TNTSEARCH_BOOLEAN=true
```

## Config scout.php
```
'tntsearch' => [
    'storage'  => storage_path(), //place where the index files will be stored
    'fuzziness' => env('TNTSEARCH_FUZZINESS', false),
    'fuzzy' => [
        'prefix_length' => 2,
        'max_expansions' => 50,
        'distance' => 2
    ],
    'asYouType' => false,
    'searchBoolean' => env('TNTSEARCH_BOOLEAN', false),
],
```

## Command
```
artisan tntsearch:import App\\MyModel    
```

## Model
```
<?php namespace App;

use Laravel\Scout\Searchable;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;
class MyModel extends Model
{
    use Searchable;

    public $asYouType = true;

    /**
     * Get the index name for the model.
     * @return string
     */
    public function searchableAs()
    {
        return 'my_index';
    }

    /**
     * Get the indexable data array for the model.
     * @return array
     */
    public function toSearchableArray()
    {
        return $this->only([
            'city',
            'state',
            'zip',
            'country',
        ]);
    }
}
```

## Example

Sanitize keywords...
```
public function search(Request $request){
    $this->validate($request, [
        'keywords' => 'required|string|min:2'
    ]);

    $keywords = $request->get('keywords');
    $keywords = str_replace('_', ' ', $keywords);
    $keywords = str_replace('-', ' ', $keywords);
    $keywords = str_replace('/', ' ', $keywords);
    $keywords = str_replace('\\', ' ', $keywords);
    $keywords = str_replace(',', ' ', $keywords);
    $keywords = str_replace('.', ' ', $keywords);
    $keywords = str_replace('`', ' ', $keywords);

    return Location::search($keywords)->take(100)->get();
}
```