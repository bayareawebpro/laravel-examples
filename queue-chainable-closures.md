## Chainable Closure (thenCall)

Currently Laravel does not support Queued Closures as children of a job chain. This helper method will
map closures to Serialized "CallQueuedClosure" class instances which allows them to be chained.

```php
use Illuminate\Support\Collection;
use Illuminate\Queue\CallQueuedClosure;
use Illuminate\Queue\SerializableClosure;

/**
 * Then Call as Queued Closure
 * @param \Closure ...$closures
 * @return array
 */
function thenCall(...$closures){
    return Collection::make($closures)
    ->map(fn(\Closure $closure) => new CallQueuedClosure(new SerializableClosure($closure)))
    ->all();
}
```

#### Usage

```php
$model = new Model;

Task::dispatch($model)->chain(thenCall(
    function() use ($model){
        $model->save();
    },
    function() use ($model){
        $model->notify(new Notification);
    }
));
```