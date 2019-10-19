# VueJS Component

```vue
<script>
    export default {
        data() {
            return {
                eventSource: null,
                isSupported: false,
                isLoading: false,
                progress: 0,
                output: [],
            }
        },
        methods: {
            run() {
                this.output = []
                this.isLoading = true
                this.eventSource = new EventSource('/event-source/demo');
                this.eventSource.addEventListener('status', event => {

                    console.info(event)
                    if (event.data) {
                        this.output.push(JSON.parse(event.data))
                        this.$nextTick(this.scrollTop)
                    }
                    if (event.lastEventId === 'close') {
                        this.closeEventSource()
                    }
                }, false)
                this.eventSource.addEventListener('error', this.closeEventSource, false)
            },
            closeEventSource() {
                this.eventSource.close()
                this.eventSource = null
                this.isLoading = false
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
            <h1>Command Name</h1>
            <hr>
            <button
                class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded"
                type="button"
                @click="run"
                :disabled="isLoading">
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
<style>
    .console-output {
        padding: 15px;
        height: 600px;
        max-height: 768px;
        overflow-y: scroll;
        background: black;
        color: #00ebff;
    }
    .console-output pre{
        padding: 5px 0;
    }
    .console-error {
        color: red;
    }
</style>
```


```php
<?php
use App\Services\EventSource;
use Illuminate\Support\Facades\Route;

Route::get('/event-source/demo', function () {

    return response()->stream(function () {
        EventSource::start('deployment', $timeout = 10000);
        EventSource::event('status', array(
            'message' => "Starting Deployment",
            'error'   => false,
        ));
        sleep(1); //Fake Process

        EventSource::event('status', array(
            'message' => "Compiling Assets",
            'error'   => false,
        ));
        sleep(1); //Fake Process

        $index = 1;
        while($index < 7){
            EventSource::event('status', array(
                'message' => "Installing Package $index...",
            ));
            sleep(1);
            EventSource::event('status', array(
                'message' => "Package $index Installed!",
            ));
            $index++;
        }
        sleep(1); //Fake Process
        EventSource::event('status', array(
            'message' => "Error Encountered...",
            'error'   => true,
        ));
        EventSource::event('status', array(
            'message' => "Package XXX failed to be installed...",
            'error'   => true,
        ));
        sleep(2); //Fake Process
        EventSource::event('status', array(
            'message' => "Deployment Failed!",
            'error'   => true,
        ));
        EventSource::close();
    }, 200, [
        'Content-Type'      => 'text/event-stream',
        'Cache-Control'     => 'no-cache',
        'X-Accel-Buffering' => 'no',
    ]);
});
```


```php
<?php declare(strict_types=1);

namespace App\Services;

/**
 * Event Source Stream
 * @source https://www.html5rocks.com/en/tutorials/eventsource/basics/
 */
class EventSource{

    public static function start($id = null, $timeout = 10000): void
    {
        echo "retry: $timeout".PHP_EOL;
        if(!is_null($id)){
            echo "id: $id".PHP_EOL;
        }
        self::flush();
    }

    public static function event($name, array $data = array()): void
    {
        echo "event: $name".PHP_EOL;
        echo "data: ".self::encode($data).PHP_EOL;
        self::flush();
    }

    public static function close(): void
    {
        echo "id: close\n".PHP_EOL;
        echo "data: \n\n\n".PHP_EOL;
        self::flush();
    }

    protected static function encode(array $data): string
    {
        return (string) json_encode($data);
    }

    public static function flush(): void
    {
        echo PHP_EOL;
        if(ob_get_level() > 0){
            ob_flush();
            flush();
        }
    }
}
```