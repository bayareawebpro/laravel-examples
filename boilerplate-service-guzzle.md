# Guzzle API Client

```php
<?php
$client = new ApiClient();
$client->setEndpoint();
$client->setTimeout();
$client->setData([...]);
$client->send();
```

```php
<?php declare(strict_types=1);

namespace App\Services;
use GuzzleHttp\Client;
use GuzzleHttp\Psr7\Response;
use GuzzleHttp\RequestOptions;
use GuzzleHttp\Exception\RequestException;

class ApiClient
{
    protected $data;
    protected $client;
    protected $endpoint;
    protected $timeout;

    public function __construct()
    {
        $this->data = [];
        $this->endpoint = null;
        $this->timeout = 300;
        $this->client = new Client([
            'connect_timeout' => 10, // Connection timeout
        ]);
    }

    /**
     * Set the API Endpoint
     * @param string $endpoint
     * @return self
     */
    public function setEndpoint(string $endpoint): self
    {
        $this->endpoint = $endpoint;
        return $this;
    }
    /**
     * Set the API Endpoint
     * @param int $timeout
     * @return self
     */
    public function setTimeout(int $timeout): self
    {
        $this->timeout = $timeout;
        return $this;
    }
    /**
     * Handle the event.
     * @param array $data
     * @return self
     */
    public function setData(array $data): self
    {
        $this->data = $data;
        return $this;
    }
    /**
     * Send Api Submission
     * @return void
     */
    public function send(): void
    {
        /** @var Response $response */
        $promise = $this->client->postAsync($this->endpoint, [
            RequestOptions::TIMEOUT     => $this->timeout,
            RequestOptions::FORM_PARAMS => $this->data,
            RequestOptions::HTTP_ERRORS => true,
            RequestOptions::VERIFY      => false,
        ]);
        $promise->then(
            function (Response $response) {
                $this->log('debug', '🚀️ API SENT', $response->getBody()->getContents());
                return $response;
            },
            function (RequestException $exception) {
                $response = '';
                if ($exception->hasResponse()) {
                    $response = $exception->getResponse()->getBody()->getContents();
                }
                $this->log('error', '🚫 API FAILED', $response);
                return $exception;
            }
        )->wait();
    }
    protected function log(string $level, string $message, string $response)
    {
        $hostname = parse_url($this->endpoint, PHP_URL_HOST);
        logger()->$level("{$message} {$hostname}", [
            'endpoint' => $this->endpoint,
            'request'  => $this->data,
            'response' => \GuzzleHttp\json_decode($response),
        ]);
    }
}
```