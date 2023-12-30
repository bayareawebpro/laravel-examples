```php

<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Contracts\Config\Repository as Config;
use Illuminate\Http\Client\PendingRequest;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Collection;

class MeiliSearch
{
    private Config $config;

    public static function make(): self
    {
        return app(MeiliSearch::class);
    }

    public function __construct(Config $config)
    {
        $this->config = $config;
    }

    public function search(string $index, string $keywords, array $filters = [], string $collect = 'hits'): Collection
    {
        return $this->getClient()->get("indexes/{$index}/search", array_merge($filters, [
            'q' => $keywords,
        ]))->collect($collect);
    }

    protected function getClient(): PendingRequest
    {
        return Http::withHeaders([
            'Authorization' => 'Bearer '.$this->config->get('scout.meilisearch.key'),
        ])->baseUrl($this->config->get('scout.meilisearch.host'));
    }

    public function getApiKeys(): Collection
    {
        return Http::withHeaders([
            'Authorization' => 'Bearer '.$this->config->get('scout.meilisearch.secret'),
        ])->baseUrl($this->config->get('scout.meilisearch.host'))->get('keys')->collect();
    }

    public function getHealthStatus(): Collection
    {
        return $this->getClient()->get('health')->collect();
    }

    public function getIndexes(): Collection
    {
        return $this->getClient()->get('indexes')->collect();
    }

    public function getIndexStats(string $index): Collection
    {
        return $this->getClient()->get("indexes/{$index}/stats")->collect();
    }

    public function getIndexTasks(string $index): Collection
    {
        return $this->getClient()->get('tasks', [
            'indexUid' => $index,
            'status' => 'succeeded',
        ])->collect('results');
    }

    public function createIndex(string $index, string $key = 'id'): Collection
    {
        return $this->getClient()->post('indexes', ['uid' => $index, 'primaryKey' => $key])->collect();
    }

    public function destroyIndex(string $index): Collection
    {
        return $this->getClient()->delete("indexes/{$index}")->collect();
    }

    public function flushIndex(string $index): Collection
    {
        return $this->getClient()->delete("indexes/{$index}/documents")->collect();
    }

    public function getIndexStatus(string $index): Collection
    {
        return $this->getClient()->get("indexes/{$index}/updates")->collect();
    }

    public function getIndexSettings(string $index): Collection
    {
        return $this->getClient()->get("indexes/{$index}/settings")->collect();
    }

    public function updateIndexSettings(string $index, array $settings): Collection
    {
        return $this->getClient()->patch("indexes/{$index}/settings", $settings)->collect();
    }

    public function updateIndexAttributes(string $index, array $settings): Collection
    {
        return $this->getClient()->post("indexes/{$index}/settings/searchable-attributes", $settings)->collect();
    }

    public function createOrReplaceDocuments(string $index, array $settings): Collection
    {
        return $this->getClient()->post("indexes/{$index}/documents", $settings)->collect();
    }

    public function createDump(): Collection
    {
        return $this->getClient()->post('dumps')->collect();
    }

    public function getDumpStatus(string $uid): Collection
    {
        return $this->getClient()->get("dumps/{$uid}/status")->collect();
    }
}

```