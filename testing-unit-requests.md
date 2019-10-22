# Unit Testing Form Requests

This is an example of testing a FormRequest that does more then validate, it merges input before 
it's validated to help users.  This test insures what's merged is reasonably accurate.

The Form Request will also load different rules from classes which are preconfigured.  
It's a multi-step form with multiple versions.  This is step one, which all version share.


#### Expectations

- It will resolve rules from configured class names.
- It can guess a result by geo-coding the primary input.
- It can auto-fill missing attributes from a local database (zip_codes).

```php
<?php declare(strict_types=1);

namespace Tests\Unit;

use App\Leads\Requests\LeadRequest;
use App\Leads\Rules\Locations;
use Tests\Mocks\PlacesMock;
use App\Location;

use Tests\TestCase;

class LeadRequestTest extends TestCase
{
    public function test_will_resolve_rules_from_configured_classes()
    {
        $resolvedRules = LeadRequest::create('/', 'get', [
            'form_step'    => 1,
            'form_type'    => "automobile",
            'form_version' => "full",
        ])->rules();

        foreach (Locations::rules() as $field => $rule) {
            $this->assertArrayHasKey($field, $resolvedRules, "Evaluated rules contain $field");
        }
    }

    public function test_it_can_autofill_attributes_from_minimal_input()
    {
        list(
            $pickupZipActual,
            $pickupCityActual,
            $pickupStateActual,
            $dropoffZipActual,
            $dropoffCityActual,
            $dropoffStateActual
            ) = $this->getMockData();

        // Simulate lazy input with minimal values (washington, dc)
        // Grab the "data to be validated" and assert expectations.
        $merged = LeadRequest::create('/', 'get', [
            'form_step'        => 1,
            'form_type'        => "automobile",
            'form_version'     => "full",
            'pickup_location'  => [
                'formatted_address' => "$pickupCityActual, $pickupStateActual",
            ],
            'dropoff_location' => [
                'formatted_address' => "$dropoffCityActual, $dropoffStateActual",
            ],
        ])->validationData();

        // Pluck the values from the merged data.
        $pickupZip = data_get($merged, 'pickup_location.zip');

        // Assert values are set.
        $this->assertNotEmpty($pickupZip, "pickup zip was found");

        // Assert value is reasonably accurate.
        $this->assertTrue((
            $this->getExistsInCity($pickupZip, $pickupCityActual, $pickupStateActual) ||
            $this->getExistsInState($pickupZip, $pickupStateActual)
        ),
        <<<MSG
        Merged pickup zip $pickupZip 
        matches $pickupZipActual 
        in $pickupCityActual, $pickupStateActual
        or $pickupStateActual
        MSG
        );

        // Pluck the values from the merged data.
        $dropoffZip = data_get($merged, 'dropoff_location.zip');

        // Assert values are set.
        $this->assertNotEmpty($dropoffZip, "dropoff zip was found");

        // Assert value is reasonably accurate.
        $this->assertTrue((
            $this->getExistsInCity($dropoffZip, $dropoffCityActual, $dropoffStateActual) ||
            $this->getExistsInState($dropoffZip, $dropoffStateActual)
        ),
        <<<MSG
        Merged dropoff zip $dropoffZip 
        matches $dropoffZipActual 
        in $dropoffCityActual, $dropoffStateActual
        or $pickupStateActual
        MSG
        );
    }

    protected function getMockData(): array
    {
        $pickup = PlacesMock::random();
        $dropoff = PlacesMock::random();
        $pickupZipActual    = data_get($pickup, 'zip');
        $pickupCityActual   = data_get($pickup, 'city');
        $pickupStateActual  = data_get($pickup, 'state_code');
        $dropoffZipActual   = data_get($dropoff, 'zip');
        $dropoffCityActual  = data_get($dropoff, 'city');
        $dropoffStateActual = data_get($dropoff, 'state_code');
        return array(
            $pickupZipActual,
            $pickupCityActual,
            $pickupStateActual,
            $dropoffZipActual,
            $dropoffCityActual,
            $dropoffStateActual,
        );
    }

    protected function getExistsInCity($pickupZip, $pickupCityActual, $pickupStateActual): bool
    {
        return Location::query()
            ->where('zip', $pickupZip)
            ->where('city', $pickupCityActual)
            ->where('state_code', $pickupStateActual)
            ->where('iso', 'US')
            ->exists();
    }

    protected function getExistsInState($pickupZip, $pickupStateActual): bool
    {
        return Location::query()
            ->where('zip', $pickupZip)
            ->where('state_code', $pickupStateActual)
            ->where('iso', 'US')
            ->exists();
    }
}
```
