# Eloquent Calendar Days

Build a calendar of entries for every day in the current month.

```php
[
    [
     "date" => "05/01/2020",
     "entries" => [...],
    ],
    [
     "date" => "05/02/2020",
     "entries" => [...],
    ],
]
```


```php
// Calendar Details
$end = now()->endOfMonth();
$start = now()->startOfMonth();
$length = now()->daysInMonth;

// Calendar Query
$pages = App\Model::query()
    ->whereBetween('updated_at', [$start, $end])
    ->take(10)
    ->get();

// Map Entries to Day
Collection::make(range(1, $length))->map(function ($days) use ($pages) {
    $now = now()->startOfMonth()->addDays($days - 1);
    return [
        'date' => $now->format('m/d/Y'),
        'entries' => $pages->filter(function ($model) use ($now) {
            return optional($model->updated_at)->isSameDay($now);
        })
    ];
});
```