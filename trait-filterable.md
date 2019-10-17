```
<?php declare(strict_types=1);

namespace App\Models\Traits;

use Illuminate\Support\Arr;
use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

trait Filterable{
    public static $log = false;

    /**
     * Searchable Attributes. (PUT THIS IN YOUR MODEL)
     * @var $searchable array
     */
    protected $searchable = [
        'id' => 'exact',
        'component' => 'exact',
        'state->description' => 'fuzzy',
        'state->title' => 'fuzzy',
        'tags' => 'relation',
        'user' => 'relation',
        'parent' => 'relation',
        'children' => 'relation',
    ];

    /**
     * Get the related model.
     * @param $model Model
     * @param $relation string
     * @return String
     */
    protected function getRelatedSearchableModel($model, $relation)
    {
        return $model->$relation()->getModel();
    }

    /**
     * Boot the Trait.
     * @param Builder $query
     * @param array $params
     * @return Builder
     */
    public function scopeSearch(Builder $query, $params = [])
    {
        /**
         * Convert Request or Array to Collection
         * @var $params Collection|Request
         */
        if(method_exists($params, 'toArray')){
            $params = $params->toArray();
        }
        /** Remove Empty Parameter Fields. */
        $searchParams = collect($params)->reject(function($value, $field){
            return !isset($value);
        });
        /** Get the searchable rules. */
        $rules = collect($this->searchable);
        /**
         * Interpolate "relation" searchable fields to their requested values.
         * The field name must match the table name!
         *
         * Request Field Format: {field_name}_{relation}
         *
         * Query String Examples:
         * tags_related=1       | numeric type  | (query: whereHas)
         * tags_related[]=1     | array type    | (query: whereHas)
         * tags_related=true    | other type    | (query: has)
         *
         * tags_unrelated=1     | numeric type  | (query: whereDoesntHave)
         * tags_unrelated[]=1   | array type    | (query: whereDoesntHave)
         * tags_unrelated=true  | other type    | (query: doesntHave)
         */
        $relations = $rules->filter(function ($value, $key) {
            return $value === 'relation';
        })
        ->mapWithKeys(function($fieldType, $field) use ($searchParams){
            foreach(["related", "unrelated"] as $relationType){
                if($value = $searchParams->get("{$field}_{$relationType}")){
                    return [
                        $field => (object) [
                            'type' => $relationType, //related || unrelated
                            'value' => $value,
                        ]
                    ];
                }
                return [];
            }
        });
        /**
         * Interpolate "exact" searchable fields to their requested values.
         * Appended directly after relation queries.
         */
        $exact = $rules
            ->filter(function ($value, $key) {
                return $value === 'exact';
            })
            ->mapWithKeys(function($type, $field) use ($searchParams){
                if($value = $searchParams->get($field)){
                    return [$field => $value];
                }
                return [];
            });
        /**
         * Interpolate "loose" searchable fields to their requested values.
         * Appended directly after "exact" queries.
         */
        $loose = $rules
            ->filter(function ($value, $key) {
                return !in_array($value, ['relation', 'exact']);
            })
            ->mapWithKeys(function($type, $field) use ($searchParams){
                if($value = $searchParams->get($field, $searchParams->get('search'))){
                    return [$field => $value];
                }
                return [];
            });
        /**
         * Query Chain
         * 1) Relation Queries.
         */
        if($relations->count()){
            /** Add Relation Queries. */
            $relations->each(function($params, $relation) use ($query, $searchParams){
                //Insure Query Value is Array
                $needsWhereIn = is_array($params->value);
                /** @var $relatedModel Model */
                $relatedModel = $this->getRelatedSearchableModel($query->getModel(), $relation);
                $relatedTable = $relatedModel->getTable();
                $relatedPrimaryKey = $relatedModel->getKeyName();
                $whereInTableField = "$relatedTable.$relatedPrimaryKey";
                switch($params->type){
                    case 'related':
                        if($needsWhereIn){
                            $query->whereHas($relation, function (Builder $query) use ($whereInTableField, $params) {
                                $query->whereIn($whereInTableField, Arr::wrap($params->value));
                            });
                            break;
                        }
                        $query->has($relation);
                        break;
                    case 'unrelated':
                        if($needsWhereIn){
                            $query->whereDoesntHave($relation, function (Builder $query) use ($whereInTableField, $params) {
                                $query->whereIn($whereInTableField, Arr::wrap($params->value));
                            });
                            break;
                        }
                        $query->doesntHave($relation);
                        break;
                }
            });
        }
        /**
         * Query Chain
         * 2) Exact Queries.
         */
        if($exact->count()){
            $query->where(function($query) use ($exact){
                $exact->each(function($value, $field) use ($query){
                    $query->where($field, $value);
                });
            });
        }
        /**
         * Query Chain
         * 3) Loose Queries.
         */
        if($loose->count()){
            $accumulated = [];
            $query->where(function($query) use ($loose, $rules, &$accumulated){
                $index = 0;
                $loose->each(function($value, $field) use (&$index, $query, $rules, &$accumulated){
                    $where = $index === 0 ? 'where' : 'orWhere';
                    switch ($rules->get($field)) {
                        case 'starts-with':
                            $query->$where($field, 'like', "%" . $value);
                            $statement = "$where $field 'like'  %$value";
                            break;
                        case 'ends-with':
                            $query->$where($field, 'like', $value . "%");
                            $statement = "$where $field 'like' $value%";
                            break;
                        case 'sounds-like':
                            $query->$where($field, 'sounds like', $value);
                            $statement = "$where $field 'sounds like' $value";
                            break;
                        default: //fuzzy
                            $query->$where($field, 'like', "%" . $value . "%");
                            $statement = "$where $field 'like' %$value%";
                            break;
                    }
                    $index++; //Increment the index.
                    if(Searchable::$log) {
                        array_push($accumulated, $statement);
                    }
                });
            });
        }
        if(Searchable::$log){
            logger()->info('SearchableTrait', array(
                'searchParams' => $searchParams->toArray(),
                'relations' => $relations->toArray(),
                'exact' => $exact->toArray(),
                'loose' => $loose->toArray(),
                'looseStatements' => isset($accumulated) ? $accumulated : null
            ));
        }
        return $query;
    }
}
```