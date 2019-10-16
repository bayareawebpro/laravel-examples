```
$files = Finder::create()->in(base_path('routes/other-routes'))->name('*.php');
Collection::make($files)->each(function(SplFileInfo $file){
    require_once $file->getRealPath();
});
```