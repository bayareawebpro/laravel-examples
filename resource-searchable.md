## Searchable JSON Resource

```php
use App\Http\Resources\SearchableResource;

//Example Request
app('request')->merge([
    'search' => 'test',  //Apply Searchable
    'sort' => 'asc', //Apply Sort
    'per_page' => 4, //Apply PerPage
    'order_by' => 'email', //Apply Order
    'settings->notify' => true, //Apply Filter
    'city' => 'Los Angeles' //Apply Filter
]);

return SearchableResource::make(User::query())
    ->filterable(['settings->notify','profile:city'])
    ->searchable(['name', 'email','profile:city'])
    ->appendable(['computedProp'])
    ->orderBy('email')
    ->sort('desc')
    ->paginate(20);
```

```php
<?php declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Foundation\Validation\ValidatesRequests;
use Illuminate\Contracts\Support\Responsable;
use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Contracts\Support\Jsonable;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Collection;
use Illuminate\Http\JsonResponse;
use Illuminate\Validation\Rule;
use Illuminate\Http\Request;
use Illuminate\Support\Str;
use Illuminate\Support\Arr;

class SearchableResource implements Responsable, Arrayable, Jsonable
{
    use ValidatesRequests;

    protected Builder $query;
    protected Request $request;

    protected int $paginate = 12;
    protected string $sort = 'desc';
    protected string $order_by = 'id';

    protected array $perPageOptions = [4, 8, 12, 16, 24, 30];
    protected array $orderable = ['id', 'created_at', 'updated_at'];
    protected array $filterable = ['id', 'created_at', 'updated_at'];

    protected array $hiddenProperties = ['data', 'first_page_url', 'last_page_url', 'next_page_url', 'prev_page_url', 'path'];

    protected array $searchable = [];
    protected array $appendable = [];

    public function validationRules()
    {
        $rules = Collection::make($this->filterable)->mapWithKeys(function ($field) {
            return [$field => ['sometimes', 'nullable']];
        });
        return $rules->merge([
            'search'   => ['sometimes', 'nullable', 'string'],
            'sort'     => ['sometimes', 'string', Rule::in(['asc', 'desc'])],
            'per_page' => ['sometimes', 'string', Rule::in($this->perPageOptions)],
            'order_by' => ['sometimes', 'string', Rule::in($this->orderable)],
        ])->toArray();
    }

    public static function make($query): self
    {
        return app(static::class, compact('query'));
    }

    public function __construct(Request $request, $query)
    {
        $this->request = $request;
        $this->query = $query;
    }

    public function filterable(array $filterable): self
    {
        $this->filterable = array_merge($this->filterable, $filterable);
        return $this;
    }

    public function appendable(array $appendable): self
    {
        $this->appendable = array_merge($this->appendable, $appendable);
        return $this;
    }

    public function searchable(array $searchable): self
    {
        $this->searchable = array_merge($this->searchable, $searchable);
        return $this;
    }

    public function orderable(array $orderable): self
    {
        $this->orderable = array_merge($this->orderable, $orderable);
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
        $this->validate($this->request ?? request(), $this->validationRules());

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
            foreach ($this->filterable as $index => $field) {
                if ($this->request->filled($field)) {
                    $keyword = $this->request->get($field);
                    if (Str::contains($field, ':')) {
                        $this->buildRelationClause($query, 'where', $field, $keyword);
                    } else {
                        $this->buildClause($query, 'where', $field, $keyword);
                    }
                }
            }
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
                        $this->buildRelationClause($query, $clause, $field, $keyword);
                    } else {
                        $this->buildClause($query, $clause, $field, $keyword);
                    }
                }
            });
        }
    }

    protected function buildClause(Builder $query, string $clause, string $field, string $keyword): void
    {
        throw_unless(
            method_exists($query, $clause),
            \InvalidArgumentException::class,
            "Invalid Clause: $clause"
        );
        $query->$clause($field, 'like', "%{$keyword}%");
    }

    protected function buildRelationClause(Builder $query, string $clause, string $field, string $keyword): void
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
        return (string)(in_array($value, $this->orderable) ? $value : $this->order_by);
    }

    protected function formatPaginator(LengthAwarePaginator $paginator): array
    {
        return array_merge(Arr::except($paginator->toArray(), $this->hiddenProperties), [
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
