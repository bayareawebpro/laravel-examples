```php
<?php
namespace App\Console\Commands;
use Illuminate\Console\Command;
use Illuminate\Foundation\Composer;
class DumpAutoload extends Command
{
    /**
     * The name and signature of the console command.
     * @example https://stackoverflow.com/questions/37238547/run-composer-dump-autoload-from-controller-in-laravel-5
     * @var string
     */
    protected $signature = 'dump-autoload';
    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Regenerate framework autoload files';
    /**
     * The Composer instance.
     *
     * @var \Illuminate\Foundation\Composer
     */
    protected $composer;
    /**
     * Create a new command instance.
     *
     * @param Composer $composer
     * @return void
     */
    public function __construct(Composer $composer)
    {
        parent::__construct();
        $this->composer = $composer;
    }
    /**
     * Execute the console command.
     *
     * @return void
     */
    public function handle()
    {
        $this->composer->dumpAutoloads();
        $this->composer->dumpOptimized();
    }
}

```
