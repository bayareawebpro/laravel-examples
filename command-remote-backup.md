### Remote Backup

Compress and stream a file to your CDN easily.

```php
<?php declare(strict_types=1);

namespace App\Console\Deployment;

use Illuminate\Console\Command;
use Illuminate\Http\File as HttpFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\File;
use Carbon\Carbon;

class DatabaseBackup extends Command
{
    const CDN_DISK = 'spaces';
    const CDN_PATH = 'backups/database';

    /**
     * The name and signature of the console command.
     * @var string
     */
    protected $signature = 'database:backup';

    /**
     * The console command description.
     * @var string
     */
    protected $description = 'Backup the database to Remote Filesystem.';

    /**
     * Create a new command instance.
     * @return void
     */
    public function __construct()
    {
        parent::__construct();
    }
    /**
     * Destruct / Delete Archive if Exists
     * @return void
     */
    public function __destruct()
    {
        File::delete($this->archivePath());
    }

    /**
     * Execute the console command.
     * @return mixed
     * @throws \Throwable
     */
    public function handle()
    {
        $this->call('telescope:clear');
        $this->backupSnapshot();
        $this->validateBackups();
    }

    /**
     * Make Archive of Snapshot.
     * @return bool
     */
    protected function makeArchive(): bool
    {
        $this->info("Creating Archive from Snapshot...");
        $zip = new \ZipArchive();
        if ($zip->open($this->archivePath(), \ZipArchive::CREATE) === TRUE) {
            $zip->addFile($this->snapshotPath());
            return $zip->close();
        }
        return false;
    }

    /**
     * Generate Archive Filename.
     * @return string
     */
    protected function archiveFilename(): string
    {
        $timestamp = Carbon::now()->toDateString();
        $environment = config('app.env', 'local');
        return "db-$environment-$timestamp.zip";
    }

    /**
     * Snapshot File Exists.
     * @return bool
     */
    protected function snapshotExists(): bool
    {
        return File::exists($this->snapshotPath());
    }

    /**
     * Get Snapshot Path.
     * @return string
     */
    protected function snapshotPath(): string
    {
        return database_path('snapshot.sql');
    }

    /**
     * Get Snapshot Path.
     * @return string
     */
    protected function archivePath(): string
    {
        return database_path('snapshot.zip');
    }

    /**
     * Validate the Remote Backups & Trim Older than 1 Month.
     * @return void
     */
    protected function validateBackups(): void
    {
        $lastMonth = Carbon::now()->subMonth();
        $disk = Storage::disk(static::CDN_DISK);

        $this->alert("Validating Backups: {$lastMonth->toDateString()}");

        foreach ($disk->allFiles(static::CDN_PATH) as $path) {
            if (Carbon::parse($disk->lastModified($path))->lt($lastMonth)) {
                $this->error("Backup Expired: $path");
                $disk->delete($path);
            } else {
                $this->info("Backup Valid: $path");
            }
        }
    }

    /**
     * Backup the Snapshot Archive.
     * @return void
     */
    protected function backupSnapshot(): void
    {
        $disk = Storage::disk(static::CDN_DISK);

        if(!$this->snapshotExists()){
            $this->error("Snapshot does not exist.");
            abort(1);
        }

        if(!$this->makeArchive()){
            $this->error("Failed to Create Archive.");
            abort(1);
        }

        $fileName = $this->archiveFilename();
        $httpFile = new HttpFile($this->archivePath());

        if (!$disk->putFileAs(static::CDN_PATH, $httpFile, $fileName)) {
            $this->info("Failed to Upload Archive: $fileName");
            abort(1);
        }

        $this->info("Archive Uploaded Successfully");
        $this->line(bytesForHumans($httpFile->getSize()));

        $this->line($disk->url($this->remoteUrl($fileName)));
    }

    /**
     * Get Remote Backup Url
     * @param string $fileName
     * @return string
     */
    protected function remoteUrl(string $fileName): string
    {
        return static::CDN_PATH."/{$fileName}";
    }
}
```