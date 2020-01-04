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

use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Foundation\Validation\ValidatesRequests;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Contracts\Support\Responsable;
use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Contracts\Support\Jsonable;
use Illuminate\Support\Arr;
use Illuminate\Support\Collection;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class SearchableResource implements Responsable, Arrayable, Jsonable
{
    use ValidatesRequests;

    protected $sort = 'desc';
    protected $paginate = 12;
    protected $order_by = 'id';
    protected $searchable = [];
    protected $appendable = [];
    protected $filterable = ['id', 'created_at', 'updated_at'];
    protected $request;

    public static function validationRules()
    {
        return [
            'search'   => 'sometimes|nullable|string|max:255',
            'sort_by'  => 'sometimes|string|in:asc,desc',
            'per_page' => 'sometimes|numeric|in:4,8,16,24',
            'order_by' => 'sometimes|string|in:id,created_at,updated_at',
        ];
    }

    /**
     * @var Builder
     */
    protected $query;

    public static function make($query): self
    {
        return app(static::class, compact('query'));
    }

    public function __construct(Request $request, $query)
    {
        $this->request = $request;
        $this->query = $query;
    }

    public function filterable($filterable): self
    {
        $this->filterable = array_merge($this->filterable, (array)$filterable);
        return $this;
    }

    public function appendable($appendable): self
    {
        $this->appendable = array_merge($this->appendable, (array)$appendable);
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

    public function toCollection(): Collection
    {
        $this->applySearchable();
        $this->applyFilterable();

        $sort = $this->getSort();
        $order = $this->getOrderBy();

        $paginator = $this->query
            ->orderBy($order, $sort)
            ->paginate($this->getPerPage());


        return Collection::make([
            'resource' => [
                'data'       => $this->formatEntries($paginator->items()),
                'query'      => $this->formatQuery($paginator, $order, $sort),
                'pagination' => $this->formatPaginator($paginator),
            ],
        ]);
    }

    protected function formatEntries(array $entries): array
    {
        if (count($this->appendable)) {
            foreach ($entries as $entry) {
                $entry->append($this->appendable);
            }
        }
        return $entries;
    }

    public function toResponse($request): JsonResponse
    {
        $this->request = $request;
        $this->validate($request, static::validationRules());
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
                    if (Str::contains($field, ':')) {
                        $this->buildRelationQuery($query, $clause, $field, $keyword);
                        return;
                    }
                    throw_unless(
                        method_exists($query, $clause),
                        \InvalidArgumentException::class,
                        "Invalid Clause: $clause"
                    );
                    $query->$clause($field, 'like', "%{$keyword}%");
                }
            });
        }
    }

    protected function buildRelationQuery($query, $clause, $field, $keyword): void
    {
        $clause = "{$clause}Has";
        throw_unless(
            method_exists($query, $clause),
            \InvalidArgumentException::class,
            "Invalid Clause: $clause"
        );
        $segments = Collection::make(explode(':', $field));
        $query->$clause($segments->first(), function (Builder $query) use ($clause, $segments, $keyword) {
            $query->where($segments->last(), 'like', "%{$keyword}%");
        });
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

    protected function formatPaginator(LengthAwarePaginator $paginator): array
    {
        return array_merge(Arr::except($paginator->toArray(), ['data']), [
            'isFirstPage' => $paginator->currentPage() === 1,
            'isLastPage'  => $paginator->currentPage() === $paginator->lastPage(),
        ]);
    }

    protected function formatQuery(LengthAwarePaginator $paginator, string $order_by, string $sort): array
    {
        return array_merge($this->request->only($this->filterable), [
            'page'     => $paginator->currentPage(),
            'order_by' => $order_by,
            'sort'     => $sort,
            'search'   => $this->request->get('search'),
        ]);
    }
}
```
