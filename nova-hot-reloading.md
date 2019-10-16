```
<?php
class FieldServiceProvider extends ServiceProvider{
    /**
     * Bootstrap the field services.
     * @return void
     */
    public function boot()
    {
        Nova::serving(function (ServingNova $event) {
            $files = app('files');
            if($files->exists(__DIR__.'/../dist/hot')){
                Nova::remoteScript('http://localhost:8080/js/field.js');
            }else{
                Nova::script('myField', __DIR__.'/../dist/js/field.js');
            }
            Nova::style('myField', __DIR__.'/../dist/css/field.css');
        });
    }
}
```