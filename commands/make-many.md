```
php artisan make:many {name}
```

```
<?php
/*
|--------------------------------------------------------------------------
| Console Routes by Dan Alvidrez
|--------------------------------------------------------------------------
*/
use Illuminate\Support\Str;
use Illuminate\Support\Arr;
use Illuminate\Support\Facades\Artisan;


$confirmed = 'yes';
$options = ['no', 'yes'];

/**
 * Make Many Application Files.
 */
Artisan::command('make:many {name}', function () use ($confirmed, $options){

    $this->alert("Making Application Files...");
    $allCommands = Artisan::all();

    $singular   = Str::ucfirst(Str::camel($this->argument('name')));
    $plural     = Str::ucfirst(Str::camel(Str::plural($this->argument('name'))));

    if($this->askWithCompletion("Tests", $options, $confirmed) === $confirmed){
        if($this->askWithCompletion("UnitTest", $options, $confirmed) === $confirmed){
            $this->call('make:test',array(
                'name' => "{$singular}Test",
                '--unit' => false
            ));
        }
        if($this->askWithCompletion("HttpTest", $options, $confirmed) === $confirmed){
            $this->call('make:test',array(
                'name' => "{$singular}Test",
                '--unit' => false
            ));
        }
        if(Arr::has($allCommands, 'dusk:make')){
            if($this->askWithCompletion("BrowserTest", $options, $confirmed) === $confirmed){
                $this->call('dusk:make',array(
                    'name' => "{$singular}Test"
                ));
            }
        }
    }
    if($this->askWithCompletion("Request", $options, $confirmed) === $confirmed) {
        $this->call('make:request', array(
            'name' => "{$plural}Request"
        ));
    }
    if($this->askWithCompletion("Controller", $options, $confirmed) === $confirmed) {
        $this->call('make:controller', array(
            "-r" => true,
            "name" => "{$plural}Controller"
        ));
    }
    if($this->askWithCompletion("JsonResource", $options, $confirmed) === $confirmed) {
        $this->call('make:resource', array(
            'name' => "{$plural}Resource"
        ));
    }
    if($this->askWithCompletion("Model", $options, $confirmed) === $confirmed) {
        $this->call('make:model', array(
            'name' => "{$singular}"
        ));
    }
    if($this->askWithCompletion("Migration", $options, $confirmed) === $confirmed){
        $this->call('make:migration',array(
            'name' => "Create{$plural}Table"
        ));
    }
    if($this->askWithCompletion("Policy", $options, $confirmed) === $confirmed){
        $this->call('make:policy',array(
            'name' => "{$singular}Policy"
        ));
    }
    if($this->askWithCompletion("Observer", $options, $confirmed) === $confirmed){
        $this->call('make:observer',array(
            'name' => "{$singular}Observer"
        ));
    }
    $this->info("Done.  Don't forget to add a route if needed.");

})->describe('MakeMany: Tests (Http, Unit, Browser), Model, Migration, FormRequest, Controller, JsonResource, Policy, Observer...');

```