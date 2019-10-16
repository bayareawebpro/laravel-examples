## Usage
```
$browser
    ->visitRoute('home')
    ->waitForText('Quote Calculator')
    ->with(new PlacesSelect('@pickup_location'), function (Browser $browser) {
        $browser->selectAndAssertFirst('houston', 'Houston, TX, USA');
    })
```

### Implementation
```
<?php

namespace Tests\Browser\Components;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Component as BaseComponent;

class PlacesSelect extends BaseComponent
{

    public $selector;

    public function __construct($selector)
    {
        $this->selector = $selector;
    }

    /**
     * Get the root selector for the component.
     * @return string
     */
    public function selector()
    {
        return '';
    }

    /**
     * Assert that the browser page contains the component.
     * @param  Browser  $browser
     * @return void
     */
    public function assert(Browser $browser)
    {
        $browser->assertVisible($this->selector);
    }

    /**
     * Get the element shortcuts for the component.
     * @return array
     */
    public function elements()
    {
        return [
            '@input' => '.pac-container .pac-item',
            '@items' => '.pac-container .pac-item',
            '@firstItem' => '.pac-container .pac-item:first-of-type',
            '@dropdown' => '.pac-container',
        ];
    }

    /**
     * Select the given date.
     * @param  \Laravel\Dusk\Browser  $browser
     * @param  mixed  $value
     * @param  mixed  $expected
     * @return void
     * @throws \Throwable
     */
    public function selectAndAssertFirst(Browser $browser, $value, $expected)
    {
        $browser
            ->type($this->selector, $value)
            ->waitFor('@items')
            ->click('@firstItem')
            ->waitUntilMissing('@dropdown')
            ->assertValue($this->selector, $expected);
    }
}

```