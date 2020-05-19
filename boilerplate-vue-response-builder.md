```php
<?php

return Vue::make([...])
    ->message('Welcome back.')
    ->emit('user:login', true)
    ->commit('user', $user)
```



```php
<?php declare(strict_types=1);

namespace App\Fluent;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Contracts\Support\Responsable;

class Vue implements Responsable
{
    protected JsonResponse $response;
    protected Request $request;
    protected array $data;
    
    /**
     * @param JsonResponse $response
     * @param Request $request
     * @param array $data
     */
    public function __construct(JsonResponse $response, Request $request, array $data = [])
    {
        $this->response = $response;
        $this->request = $request;
        $this->data = $data;
    }
    
    /**
     * @param array $data
     * @return $this
     */
    public static function make(array $data):self
    {
        return app(static::class, compact('data'));
    }
    

    /**
     * @param string $key
     * @param mixed $value
     * @return $this
     */
    public function commit(string $key, $value)
    {
        data_set($this->data, "commit.{$key}", $value);
        return $this;
    }
    
    /**
     * @param string $key
     * @param mixed $value
     * @return $this
     */
    public function emit(string $key, $value)
    {
        data_set($this->data, "events.{$key}", $value);
        return $this;
    }
    
    /**
     * @param string $value
     * @return $this
     */
    public function message(string $value)
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
        $this->request = $request ?? $this->request;
        $this->response->setData($this->data);
        return $this->response->prepare($request);
    }
}
```
