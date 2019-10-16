```
<?php 
namespace Deployer;

/**
 * Custom Command Example
 */
task('artisan:reset', function (){
    run("{{bin/php}} {{release_path}}/artisan reset");
})->onStage('production');
```