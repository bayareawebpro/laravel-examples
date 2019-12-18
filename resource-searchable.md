## Searchable JSON Resource

```php
use App\Http\Resources\SearchableResource;

app('request')->merge([
    'search' => 'test',
    'sort' => 'asc',
    'per_page' => 4,
    'order_by' => 'email',
    'settings->notify' => true,
    'settings->digest' => true
]);

SearchableResource::make(User::query())
    ->filterable(['settings->notify', 'settings->digest'])
    ->searchable(['name', 'email'])
    ->orderBy('email')
    ->sort('desc')
    ->paginate(20)
    ->toArray();
```

```php
<?php declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Contracts\Support\Responsable;
use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Contracts\Support\Jsonable;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Collection;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class SearchableResource implements Responsable, Arrayable, Jsonable
{
    protected $sort = 'asc';
    protected $paginate = 8;
    protected $order_by = 'created_at';
    protected $filterable = [];
    protected $searchable = [];
    protected $request;
    protected $query;

    public static function make(Builder $query): self
    {
        return app(static::class, compact('query'));
    }

    public function __construct(Request $request, Builder $query)
    {
        $this->request = $request;
        $this->query = $query;
    }

    public function filterable($filterable): self
    {
        $this->filterable = array_merge($this->filterable, (array)$filterable);
        return $this;
    }

    public function searchable($searchable): self
    {
        $this->searchable = array_merge($this->searchable, (array)$searchable);
        return $this;
    }

    public function paginate(int $paginate): self
    {
        $this->paginate = $paginate;
        return $this;
    }

    public function orderBy(string $orderBy): self
    {
        $this->order_by = $orderBy;
        return $this;
    }

    public function sort(string $sort): self
    {
        $this->sort = $sort;
        return $this;
    }

    protected function applyFilterable(): void
    {
        $this->query->where(function (Builder $query) {
            $query->where($this->request->only($this->filterable));
        });
    }

    protected function applySearchable(): void
    {
        if ($this->request->filled('search')) {
            $this->query->where(function (Builder $query) {
                $keyword = $this->request->get('search');
                foreach ($this->searchable as $index => $field) {
                    $clause = $index === 0 ? 'where' : 'orWhere';
                    $query->$clause($field, 'like', "%{$keyword}%");
                }
            });
        }
    }

    protected function getPerPage(): int
    {
        $value = $this->request->get('per_page', $this->paginate);
        return (int)(in_array($value, range(1, 60)) ? $value : $this->paginate);
    }

    protected function getSort(): string
    {
        $value = $this->request->get('sort', $this->sort);
        return (string)(in_array($value, ['asc', 'desc']) ? $value : $this->sort);
    }

    protected function getOrderBy(): string
    {
        $value = $this->request->get('order_by', $this->order_by);
        return (string)(in_array($value, $this->getAllowedFields()) ? $value : $this->order_by);
    }

    protected function getAllowedFields(): array
    {
        return array_merge(
            $this->searchable,
            $this->filterable,
            [$this->order_by]
        );
    }

    public function toCollection(): Collection
    {
        $this->applySearchable();
        $this->applyFilterable();

        $sort = $this->getSort();
        $order_by = $this->getOrderBy();

        $paginator = Collection::make($this->query
            ->orderBy($order_by, $sort)
            ->paginate($this->getPerPage())
        );

        return Collection::make([
            'data'       => $paginator->get('data'),
            'query'      => $this->formatQuery($paginator, $order_by, $sort),
            'pagination' => $this->formatPagination($paginator),
        ]);
    }

    public function toResponse($request): JsonResponse
    {
        $this->request = $request;
        return JsonResponse::create($this->toArray());
    }

    public function toJson($options = 0): string
    {
        return $this->toCollection()->toJson($options);
    }

    public function toArray(): array
    {
        return $this->toCollection()->toArray();
    }

    protected function formatPagination(Collection $paginator): array
    {
        return array_merge($paginator->except('data')->all(), [
            'isFirstPage' => $paginator->get('current_page') === 1,
            'isLastPage'  => $paginator->get('current_page') === $paginator->get('last_page'),
        ]);
    }

    protected function formatQuery(Collection $paginator, string $order_by, string $sort): array
    {
        return array_merge($this->request->only($this->filterable), [
            'search'   => $this->request->get('search'),
            'page'     => $paginator->get('current_page'),
            'order_by' => $order_by,
            'sort'     => $sort,
        ]);
    }
}
```
