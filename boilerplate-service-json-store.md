```
$json = Json::make(storage_path('framework/testing/json-test.json'));
$json->data->put('key', true);
$json->save()
```

## Json Store
```
<?php declare(strict_types=1);

namespace App;

use Illuminate\Support\Collection;
use Illuminate\Filesystem\Filesystem;
use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Contracts\Support\Jsonable;

class Json implements Arrayable, Jsonable
{

    /**
     * @var Filesystem
     */
    protected $files;

    /**
     * @var Collection
     */
    public $data;

    /**
     * @var string
     */
    public $path;

    /**
     * Json constructor.
     * @param Filesystem $files
     * @param string $path
     * @param array $data
     */
    public function __construct(
        Filesystem $files,
        ?string $path = null,
        ?array $data = null
    )
    {
        $this->path = $path;
        $this->files = $files;
        $this->data = Collection::make($data ? $data : $this->read());
    }

    /**
     * @param string|null $path
     * @param array|null $data
     * @return static
     */
    public static function make(?string $path = null, ?array $data = null): self
    {
        return app(self::class, compact('path', 'data'));
    }

    /**
     * Read Current State from Disk.
     * @return array
     */
    protected function read(): array
    {
        try {
            if ($this->files->exists($this->path)) {
                return json_decode($this->files->get($this->path), true);
            }
        } catch (\Throwable $exception) {
            logger()->critical($exception->getMessage(), $exception->getTrace());
        }
        return [];
    }

    /**
     * Save Current State to Disk.
     * @param bool $overwrite
     * @param int $options
     * @return $this
     */
    public function save($overwrite = false, $options = JSON_FORCE_OBJECT): self
    {
        if (!$overwrite) {
            $this->mergeWithPersisted();
        }
        $this->files->put($this->path, $this->data->toJson($options));
        return $this;
    }

    /**
     * Merge Persisted Data.
     * @return $this
     */
    public function mergeWithPersisted(): self
    {
        $this->data = $this->data::make($this->read())->merge($this->data);
        return $this;
    }

    /**
     * Return the array representation of the data.
     * @return array
     */
    public function toArray(): array
    {
        return $this->data->toArray();
    }

    /**
     * Return the json representation of the data.
     * @param int $options
     * @return string
     */
    public function toJson($options = JSON_FORCE_OBJECT): string
    {
        return $this->data->toJson($options);
    }
}
```

## Unit Tests

```
<?php
namespace Tests\Unit;

use App\Json;
use Tests\TestCase;

class JsonStoreTest extends TestCase
{
    protected function getFilePath(): string
    {
        return storage_path('framework/testing/json-test.json');
    }

    protected function withState($state = null)
    {
        if (file_exists($this->getFilePath())) {
            unlink($this->getFilePath());
            $this->assertFileNotExists($this->getFilePath());
        }
        if(!is_null($state)){
            file_put_contents(storage_path('framework/testing/json-test.json'), $state);
            $this->assertFileExists($this->getFilePath());
            $this->assertSame($state, file_get_contents($this->getFilePath()));
        }
    }

    protected function getInstance(): Json
    {
        return Json::make($this->getFilePath());
    }

    public function test_default_state()
    {
        $this->withState();
        $json = $this->getInstance();
        $this->assertSame([], $json->toArray());
    }

    public function test_default_loaded_state()
    {
        $this->withState('{}');
        $json = $this->getInstance();
        $this->assertSame([], $json->toArray());
    }

    public function test_can_restore_values()
    {
        $this->withState('{"testA":1, "testB":2}');
        $this->assertSame([
            "testA" => 1,
            "testB" => 2,
        ], $this->getInstance()->toArray());
    }

    public function test_can_modify_values_and_restore()
    {
        $this->withState('{"testA":1, "testB":2}');

        $json = $this->getInstance();

        $json->data->put('testC', 3);

        $this->assertSame([
            "testA" => 1,
            "testB" => 2,
            "testC" => 3,
        ], $json->toArray());

        $json->save();

        $this->assertSame('{"testA":1,"testB":2,"testC":3}', file_get_contents($this->getFilePath()));

        $this->assertSame([
            "testA" => 1,
            "testB" => 2,
            "testC" => 3,
        ], $this->getInstance()->toArray());
    }

    public function test_will_merge_state_or_overwrite(){
        $this->withState('{}');

        $jsonA = $this->getInstance();
        $jsonB = $this->getInstance();
        $jsonC = $this->getInstance();
        $jsonD = $this->getInstance();

        //Merge to disk.
        $jsonA->data->put('testA', 1);
        $jsonA->save();

        //Merge to disk.
        $jsonB->data->put('testB', 2);
        $jsonB->save();

        //Merge to disk.
        $jsonC->data->put('testC', 3);
        $jsonC->save();

        //Merge to disk.
        $jsonD->data->put('testD', 4);
        $jsonD->save();

        $this->assertSame('{"testA":1,"testB":2,"testC":3,"testD":4}', file_get_contents($this->getFilePath()));

        //Forget Everything and Overwrite
        $jsonD->data->forget(['testA', 'testB', 'testC', 'testD']);
        $jsonD->save(true);
        $this->assertSame('{}', file_get_contents($this->getFilePath()));
    }
}
```