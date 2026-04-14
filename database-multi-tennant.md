## Manager

```php
<?php declare(strict_types=1);

namespace App\Tenant;

use App\Models\User;
use Illuminate\Config\Repository;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Artisan;
use Illuminate\Database\Eloquent\Model;
use \Illuminate\Database\DatabaseManager;
use Illuminate\Database\ConnectionInterface;
use Illuminate\Contracts\Filesystem\Filesystem;

class Manager
{
    protected User $user;
    protected Repository $config;
    protected DatabaseManager $database;

    protected string|int $tenant_id;
    protected string|null $partition = null;
    protected string $connectionName = "tenant";
    protected string $defaultDatabase = "tenant";
    protected string $defaultConnection = "mysql";

    public function __construct(
        string|int      $key,
        Repository      $config,
        DatabaseManager $database,
    )
    {
        $this->tenant_id = $key;
        $this->partition = "tenant_{$key}";
        $this->database = $database;
        $this->config = $config;
    }

    public static function fromModel(Model $model): self
    {
        return app(static::class, [
            'key' => $model->getKey()
        ]);
    }

    public static function tenant(string $key): self
    {
        return app(static::class, [
            'key' => $key
        ]);
    }

    public function connect(bool $default = false): self
    {
        $this->config->set('database.connections.tenant.database', $this->partition);
        if ($default) {
            $this->config->set('database.default', $this->connectionName);
        }
        $this->database->reconnect($this->connectionName);
        $this->database->reconnect($this->defaultConnection);
        return $this;
    }

    public function disconnect(): self
    {
        $this->config->set('database.default', $this->defaultConnection);
        $this->config->set('database.connections.tenant.database', $this->defaultDatabase);
        $this->database->reconnect($this->connectionName);
        $this->database->reconnect($this->defaultConnection);
        return $this;
    }

    public function database(): ConnectionInterface
    {
        return $this->database->connection($this->connectionName);
    }

    public function databaseExists(): bool
    {
        return count($this->database->select(<<<SQL
        SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME = '{$this->partition}'
        SQL)) === 1;
    }

    public function createDatabase(): self
    {
        $charset = $this->config->get("database.connections.mysql.charset", 'utf8');
        $collate = $this->config->get("database.connections.mysql.collation", 'utf8_unicode_ci');
        $this->database->statement("CREATE DATABASE {$this->partition} CHARACTER SET {$charset} COLLATE {$collate}");
        return $this;
    }

    public function dropDatabase(): self
    {
        $this->database->statement("DROP DATABASE {$this->partition}");
        return $this;
    }

    public function migrateDatabase(): self
    {
        Artisan::call('migrate:fresh', [
            '--realpath' => database_path('migrations/tenant'),
            '--database' => $this->connectionName,
            '--force'    => true,
        ]);
        return $this;
    }

    public function hasStoragePartition(): bool
    {
        return $this->rootStorage()->exists($this->partition);
    }

    public function makeStoragePartition(): bool
    {
        return $this->rootStorage()->makeDirectory($this->partition);
    }

    public function deleteStoragePartition(): bool
    {
        return $this->rootStorage()->deleteDirectory($this->partition);
    }

    public function storage(): Filesystem
    {
        if(!$this->hasStoragePartition()){
            $this->makeStoragePartition();
        }
        return Storage::build([
            'driver' => 'local',
            'root' => storage_path("tenants/{$this->partition}"),
        ]);
    }

    public function rootStorage(): Filesystem
    {
        return Storage::build([
            'driver' => 'local',
            'root' => storage_path("tenants"),
        ]);
    }

    public function storagePath(string $path): string
    {
        return storage_path("tenants/{$this->partition}/{$path}");
    }

    public function connectionName(): string
    {
        return $this->connectionName;
    }

    public function partitionName(): string
    {
        return $this->partition;
    }

    public function tenantId(): string|int
    {
        return $this->tenant_id;
    }
}
```

## Middleware
```php
<?php declare(strict_types=1);

namespace App\Tenant\Middleware;

use Closure;
use App\Tenant\Manager;

class Multitenancy
{
    public function handle($request, Closure $next)
    {
        if (!$request->user()) {
            return $next($request);
        }
		
		// Subdomain:
        Manager::tenant($request->route('subdomain'))->connect(true);

		// Or using model key:
        Manager::fromModel($request->user())->connect(true);

        return $next($request);
    }
}
```


## Usage

```php
use App\Tenant\Middleware\Multitenancy;

Route::domain('{subdomain}.artisan.local')
	->middleware([Multitenancy::class])
	->group(function(){
	// Tennant Routes
});
```

## Tests

```php
<?php namespace Tests\Unit;

use App\Models\User;
use Tests\TestCase;
use App\Tenant\Manager;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\DB;

class TenantManagerTest extends TestCase
{
    public function test_can_use_database_connection()
    {
        $tenant = Manager::tenant('test')->connect();

        $this->assertInstanceOf(Manager::class, $tenant, 'Resolved Tenant Manager Class');

        if ($tenant->databaseExists()) {
            $tenant->dropDatabase();
        }
        $this->assertFalse($tenant->databaseExists(), 'Database Does Not Exist');

        $tenant->createDatabase();
        $this->assertTrue($tenant->databaseExists(), 'Database Created');

        $tenant->migrateDatabase();
        $this->assertTrue($tenant->database()->table('migrations')->count() > 0, 'Migrations Ran');

        $tenant->dropDatabase();
        $this->assertFalse($tenant->databaseExists(), 'Database destroyed');
        $tenant->disconnect();
    }

    /**
     * Default database connection is switched to tenant and back to default.
     */
    public function test_can_swap_default_database_connection()
    {
        $tenant = Manager::tenant('test')->connect(true);

        $this->assertSame('tenant', DB::getDefaultConnection());
        $this->assertSame('tenant', User::query()->getConnection()->getName());

        $tenant->disconnect();

        $this->assertSame('mysql', DB::getDefaultConnection());
        $this->assertSame('mysql', User::query()->getConnection()->getName());
    }

    /**
     * Directory created (storage/tenants/tenant_1)
     */
    public function test_can_manage_storage_partitions()
    {
        $tenant = Manager::tenant('test')->connect();

        $tenant->deleteStoragePartition();
        $this->assertFalse($tenant->hasStoragePartition(), 'Storage Partition Directory Does Not Exist');

        $tenant->makeStoragePartition();
        $this->assertTrue($tenant->hasStoragePartition(), 'Storage Partition Directory Created');

        $tenant->deleteStoragePartition();
        $this->assertFalse($tenant->hasStoragePartition(), 'Storage Partition Directory Does Not Exist');
    }

    /**
     * File created (storage/tenants/tenant_1/test.txt)
     */
    public function test_can_provide_scoped_storage_instance()
    {
        $user = User::factory()->create();
        $file = UploadedFile::fake()->create('test.txt');
        $tenant = Manager::fromModel($user)->connect();

        $path = $tenant->storage()->putFileAs("imports", $file, $file->getFilename());

        $this->assertFileExists($tenant->storagePath($path), 'File Exists in Tenant Storage');
        $tenant->deleteStoragePartition();
    }
}
```
