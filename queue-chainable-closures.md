## Chainable Closure

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
function queueClosures(...$closures){
    return Collection::make($closures)
    ->map(fn(\Closure $closure) => new CallQueuedClosure(new SerializableClosure($closure)))
    ->all();
}
```

#### Usage

```php
$model = Model::create(...);

Task::dispatch($model)->chain(queueClosures(
    function() use ($model){
        $model->update(['processed' => true]);
    },
    function() use ($model){
        $model->notify(new Notification);
    }
));
```
