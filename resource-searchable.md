## Searchable JSON Resource

```php
use App\Http\Resources\SearchableResource;

app('request')->merge([
    'search' => 'test',
    'settings->notify' => true
]);

return SearchableResource::make(User::query())
    ->filterable(['settings->notify'])
    ->searchable(['name', 'email'])
    ->orderBy('name')
    ->sort('desc')
    ->paginate(10)
    ->toArray();
```
```php
[
     "data" => [
       [
         "id" => 2,
         "name" => "test@provisioner.test",
         "email" => "test@provisioner.test",
         "email_verified_at" => "2019-12-17 07:55:19",
         "settings" => [
           "digest" => true,
           "notify" => true,
         ],
         "created_at" => "2019-12-17 07:55:19",
         "updated_at" => "2019-12-17 07:55:19",
       ],
     ],
     "current_page" => 1,
     "last_page" => 1,
     "per_page" => 3,
     "total" => 1,
     "orderBy" => "name",
     "sort" => "desc",
     "search" => "test",
     "isFirstPage" => true,
     "isLastPage" => true,
     "isPaginated" => false,
     "isSearching" => true,
     "isFiltering" => [
       "settings->notify" => true,
     ],
   ];
```

```php
<?php declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Contracts\Support\Jsonable;
use Illuminate\Contracts\Support\Responsable;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Collection;
use Illuminate\Http\Request;

class SearchableResource implements Responsable, Arrayable, Jsonable
{
    protected $filterable = [];
    protected $searchable = [];
    protected $paginate = 8;
    protected $orderBy = 'created_at';
    protected $sort = 'asc';
    protected $request;
    protected $query;

    public static function make(Builder $query)
    {
        return app(static::class, compact('query'));
    }

    public function __construct(Request $request, Builder $query)
    {
        $this->request = $request;
        $this->query = $query;
    }

    public function toResponse($request)
    {
        $this->request = $request;
        return JsonResponse::create($this->toArray());
    }

    public function toJson($options = 0)
    {
        return $this->toCollection()->toJson($options);
    }

    public function toArray()
    {
        return $this->toCollection()->toArray();
    }

    public function toCollection(): Collection
    {
        $this->applySearchable();
        $this->applyFilterable();

        $perPage = $this->getPerPage();
        $orderBy = $this->getOrderBy();
        $sort = $this->getSort();

        $paginator = Collection::make($this->query->orderBy($orderBy, $sort)->paginate($perPage));

        return $paginator
            ->merge(compact('orderBy', 'sort'))
            ->merge($this->request->all('search'))
            ->put('isFirstPage', $paginator->get('current_page') === 1)
            ->put('isLastPage',  $paginator->get('current_page') === $paginator->get('last_page'))
            ->put('isPaginated', $paginator->get('isFirstPage') !== $paginator->get('isLastPage'))
            ->put('isFiltering',$this->request->all($this->filterable))
            ->put('isSearching', $this->request->filled('search'))
            ->forget([
                'first_page_url',
                'prev_page_url',
                'next_page_url',
                'last_page_url',
                'path',
                'from',
                'to',
            ]);
    }

    public function filterable($filterable)
    {
        $this->filterable = array_merge($this->filterable, (array)$filterable);
        return $this;
    }

    public function searchable($searchable)
    {
        $this->searchable = array_merge($this->searchable, (array)$searchable);
        return $this;
    }

    public function paginate(int $paginate)
    {
        $this->paginate = $paginate;
        return $this;
    }

    public function orderBy(string $orderBy)
    {
        $this->orderBy = $orderBy;
        return $this;
    }

    public function sort(string $sort)
    {
        $this->sort = $sort;
        return $this;
    }

    protected function getPerPage()
    {
        $value = $this->request->get('per_page', 4);
        return in_array($value, range(1, 100, 4)) ? $value : $this->paginate;
    }

    protected function getSort()
    {
        $value = $this->request->get('sort', $this->sort);
        return in_array($value, ['asc', 'desc']) ? $value : $this->sort;
    }

    protected function getOrderBy()
    {
        $value = $this->request->get('orderBy', $this->orderBy);
        return in_array($value, $this->filterable) ? $value : $this->orderBy;
    }

    protected function applyFilterable(): void
    {
        $this->query->where(function (Builder $query) {
            $index = 0;
            foreach ($this->filterable as $field) {
                if ($this->request->filled($field)) {
                    $clause = $index === 0 ? 'where' : 'orWhere';
                    $query->$clause($field, $this->request->get($field));
                    $index++;
                }
            }
        });
    }

    protected function applySearchable(): void
    {
        if($this->request->filled('search')){
            $this->query->where(function (Builder $query) {
                $keyword=$this->request->get('search');
                foreach ($this->searchable as $index => $field) {
                    $clause = $index === 0 ? 'where' : 'orWhere';
                    $query->$clause($field, 'like', "%{$keyword}%");
                }
            });
        }
    }
}

```
