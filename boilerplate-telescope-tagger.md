# Telescope Tagger

Keeping your service providers clean is a core concept.  Here's a boilerplate 
for setting up tagger classes that tuck away your logic and tag resolution methods.

### Usage
```php
<?php
namespace App\Providers;

use App\Telescope\LeadTagger;
use Laravel\Telescope\Telescope;
use Illuminate\Support\Facades\Gate;
use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\TelescopeApplicationServiceProvider;

class TelescopeServiceProvider extends TelescopeApplicationServiceProvider
{

    /**
     * Register any application services.
     * @return void
     */
    public function register()
    {
        //...

        Telescope::filterBatch(function (Collection $entries) {
            if ($this->app->environment('local')) {
                return true;
            }
            return $entries->contains(function (IncomingEntry $entry) {
                return (
                    $entry->isFailedJob() ||
                    $entry->hasMonitoredTag() ||
                    $entry->isScheduledTask() ||
                    $entry->isReportableException() ||
                    LeadTagger::canTag($entry) // Evaluate Entry
                );
            });
        });

        Telescope::tag(function (IncomingEntry $entry) {
             // Evaluate Entry
            if (LeadTagger::canTag($entry)) {
                // Provide Tags
                return LeadTagger::getTags($entry);
            }
            return [];
        });
    }
    /**
     * Prevent sensitive request details from being logged by Telescope.
     * @return void
     */
    protected function hideSensitiveRequestDetails()
    {
        //...
    }

    /**
     * Register the Telescope gate.
     * This gate determines who can access Telescope in non-local environments.
     * @return void
     */
    protected function gate()
    {
        //...
    }
}

```

### Implementation
```php
<?php declare(strict_types=1);

namespace App\Telescope;

use Illuminate\Support\Str;
use Laravel\Telescope\IncomingEntry;

class LeadTagger
{
    const TAGS = ['lead'];

    /**
     * Is Taggable
     * @param IncomingEntry $entry
     * @return bool
     */
    public static function canTag(IncomingEntry $entry)
    {
        if ($entry->type === 'request' && $entry->content['method'] !== "GET") {
            return Str::contains($entry->content['uri'], 'quote');
        }
    }

    /**
     * Get Taggable Tags
     * @param IncomingEntry $entry
     * @return array
     */
    public static function getTags(IncomingEntry $entry)
    {
        return static::TAGS;
    }
}
```
