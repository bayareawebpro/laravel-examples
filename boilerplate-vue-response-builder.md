```php
<?php declare(strict_types=1);

namespace App\Fluent;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Contracts\Support\Responsable;

class ApiResponse implements Responsable
{
    protected $response;
    protected $request;
    protected $data;
    public function __construct(JsonResponse $response, Request $request)
    {
        $this->response = $response;
        $this->request = $request;
        $this->data = [];
    }
    /**
     * @param $key
     * @param $value
     * @return $this
     */
    public function commit($key, $value)
    {
        data_set($this->data, "commit.{$key}", $value);
        return $this;
    }
    /**
     * @param $key
     * @param $value
     * @return $this
     */
    public function event($key, $value)
    {
        data_set($this->data, "events.{$key}", $value);
        return $this;
    }
    /**
     * @param $value
     * @return $this
     */
    public function message($value)
    {
        data_set($this->data, "message", $value);
        return $this;
    }
    /**
     * @param Request $request
     * @return JsonResponse|\Illuminate\Http\Response
     */
    public function toResponse($request)
    {
        $this->response->setData($this->data);
        return $this->response->prepare($request);
    }
}
```
