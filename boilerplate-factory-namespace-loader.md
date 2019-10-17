## Factory Pattern for Resolving Many Classes from Different Namespaces.

Use case: need to resolve many different classes from one entry point without importing any classes.

```
$entityFactory = EntityFactory::fromNamespace('App\Factories');

$car = $entityFactory->make('car', ['name' => 'Sedan']);
$car->drive();

$horse = $entityFactory->make('horse', ['name' => 'Stallion']);
$horse->ride();
```

## Factory Class
```
<?php declare(strict_types=1);

namespace App\Factories;
use Illuminate\Support\Str;

class EntityFactory{
    
    private $namespace;
    
    public function __construct($namespace = '\\App\\Entity')
    {
        $this->namespace = $namespace;
    }
    
    public static function fromNamespace(string $namespace): EntityFactory
    {
        return app(self::class, compact('namespace'));
    }
    
    public function make(string $entity, array $attributes = [])
    {
        return app($this->namespace."\\".Str::studly($entity), $attributes);
    }
}

```

## Example Classes
```
namespace App\Models;
class Car{
    
    public function __construct(string $name){
        $this->name = $name;
    }
    
    public function drive(){
        echo "{$this->name}: VROOOM! VROOOM!".PHP_EOL;
    }
}
class Horse{
    
    public function __construct(string $name){
        $this->name = $name;
    }
    
    public function ride(){
        echo "{$this->name}: CLOP CLOP CLOP...".PHP_EOL;
    }
}
```