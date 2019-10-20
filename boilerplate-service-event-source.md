## Usage
- Example: https://imgur.com/a/dbg9AAE

```php
<?php
Route::get('/event-source/demo', function () {

    return EventSource::make('deployment', 10000, function(EventSource $event){
        $event->send('status', array(
            'message' => "Starting Deployment",
            'error'   => false,
        ));
        sleep(1); //Fake Process

        $event->send('status', array(
            'message' => "Compiling Assets",
            'error'   => false,
        ));
        sleep(1); //Fake Process

        $index = 1;
        while($index < 7){
            $event->send('status', array(
                'message' => "Installing Package $index...",
            ));
            sleep(1);
            $event->send('status', array(
                'message' => "Package $index Installed!",
            ));
            $index++;
        }
        sleep(1); //Fake Process
        $event->send('status', array(
            'message' => "Error Encountered...",
            'error'   => true,
        ));
        $event->send('status', array(
            'message' => "Package XXX failed to be installed...",
            'error'   => true,
        ));
        sleep(2); //Fake Process
        $event->send('status', array(
            'message' => "Deployment Failed!",
            'error'   => true,
        ));
    });
});
```

--- 

## EventSource Class

```php
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Contracts\Support\Responsable;
use Symfony\Component\HttpFoundation\StreamedResponse;

/**
 * Event Source Stream
 * @source https://www.html5rocks.com/en/tutorials/eventsource/basics/
 * @source https://dev.to/mesadhan/event-stream-server-send-events-5afk
 */
class EventSource implements Responsable
{

    protected $id;
    protected $timeout;
    public $response;

    public function __construct(string $id, int $retryTimeout, \Closure $closure)
    {
        $this->id = $id;
        $this->timeout = $retryTimeout;
        $this->response = new StreamedResponse(function () use ($closure){
            $this->start();
            $closure($this);
            $this->close();
        });
        $this->response->headers->set('Content-Type', 'text/event-stream');
        $this->response->headers->set('X-Accel-Buffering', 'no');
        $this->response->headers->set('Cach-Control', 'no-cache');
    }

    public static function make(string $id, int $retryTimeout, \Closure $closure)
    {
        return new self($id, $retryTimeout, $closure);
    }

    public function start(): void
    {
        ini_set('max_execution_time', "{$this->timeout}");
        echo "retry: {$this->timeout}" . PHP_EOL;
        echo "id: {$this->id}" . PHP_EOL;
        $this->flush();
    }

    public function send(string $name, array $data = array()): void
    {
        echo "event: $name" . PHP_EOL;
        echo "data: " . $this->encode($data) . PHP_EOL;
        $this->flush();
    }

    public function close(): void
    {
        echo "id: close\n" . PHP_EOL;
        echo "data: \n\n\n" . PHP_EOL;
        $this->flush();
    }

    protected function encode(array $data): string
    {
        return (string)json_encode($data);
    }

    protected function flush(): void
    {
        echo PHP_EOL;
        if (ob_get_level() > 0) {
            ob_flush();
            flush();
        }
    }

    public function toResponse($request): StreamedResponse
    {
        return $this->response;
    }
}
```
--- 

## VueJS Component

```vue
<script>
    export default {
        name: 'EventSource',
        data() {
            return {
                output: [],
                isLoading: false,
                isSupported: false,
            }
        },
        methods: {
            run() {
                this.output = []
                this.isLoading = true
                this.$options.stream = new EventSource('/event-source');
                this.$options.stream.addEventListener('error', this.close, false)
                this.$options.stream.addEventListener('status', this.record, false)
            },
            record(event){
                if (event.data) {
                    this.output.push(JSON.parse(event.data))
                    this.$nextTick(this.scrollTop)
                }
                if (event.lastEventId === 'close') {
                    this.close()
                }
            },
            close() {
                this.isLoading = false
                this.$options.stream.close()
                this.$options.stream = null
            },
            scrollTop() {
                if (this.$refs.console) {
                    this.$refs.console.scrollTop = this.$refs.console.scrollHeight
                }
            },
        },
        created() {
            this.isSupported = 'EventSource' in window
        }
    }
</script>
<template>
    <div>
        <div v-if="isSupported">
            <button
                @click="run"
                type="button"
                :disabled="isLoading"
                class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded">
                Run Command
            </button>
            <hr>
            <div class="console-output" ref="console">
                <pre v-for="(entry) in output" :class="{'console-error': entry.error}">{{ entry.message }}</pre>
            </div>
        </div>
        <div v-else>
            "EventStream" is not supported by this browser. Time to upgrade?
        </div>
    </div>
</template>
<style lang="sass">
    .console-output
        padding: 15px
        height: 600px
        max-height: 768px
        overflow-y: scroll
        background: black
        color: #00ebff
        pre
            padding: 5px 0
            &.console-error
                color: red
</style>
```
