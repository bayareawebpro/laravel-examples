## Configurable Trait
- https://github.com/bayareawebpro/laravel-examples/blob/master/model-trait-configurable.md

## Routes
```
<?php use Illuminate\Support\Facades\Route;

Route::group([
    'middleware' => 'auth',
    'prefix' => 'settings',
    ], function () {
    Route::get('edit', [
        'uses' => 'Account\SettingsController@edit',
        'as'   => 'settings.edit',
    ]);
    Route::put('update', [
        'uses' => 'Account\SettingsController@update',
        'as'   => 'settings.update',
    ]);
});
```

## Account Settings Controller
```
<?php declare(strict_types=1);

namespace App\Http\Controllers\Account;

use Illuminate\Http\Request;
use App\Http\Requests\SettingsRequest;
use App\Http\Controllers\Controller;

class SettingsController extends Controller
{
    /**
     * Show Edit Form.
     * @param Request $request
     * @return \Illuminate\View\View
     */
    public function edit(Request $request)
    {
        return view('account.settings', [
            'user' => $request->user(),
        ]);
    }

    /**
     * @param Request $request
     * @return \Illuminate\Http\Response
     * @throws \Illuminate\Validation\ValidationException
     */
    public function update(SettingsRequest $request)
    {
        return response([
            'user' => tap($request->user())->update([
                'settings' => $request->validated()
            ]),
        ]);
    }
}

```

## Settings Request
```
<?php

namespace App\Http\Requests;

use App\User;
use Illuminate\Support\Facades\Gate;
use Illuminate\Foundation\Http\FormRequest;

class SettingsRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     * @return bool
     */
    public function authorize()
    {
        return true;  // Gate::forUser($this->user());
    }

    /**
     * Get the validation rules that apply to the request.
     * @return array
     */
    public function rules()
    {
        return [
            'notify' => 'boolean',
            'digest' => 'boolean',
        ];
    }
}
```