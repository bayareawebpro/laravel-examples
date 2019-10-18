
## Route
```
<?php
use Illuminate\Support\Facades\Route;

Route::group([
    'prefix' => 'media',
    'middleware' => 'auth',
], function () {
    Route::post('upload', [
        'uses' => 'Media\MediaController@store',
        'as'   => 'media.store',
    ]);
});

```

## Controller
- Example for returning a file path to your SPA for joining with a model to be saved.

```
<?php declare(strict_types=1);

namespace App\Http\Controllers\Media;

use App\Http\Requests\MediaRequest;
use App\Http\Controllers\Controller;

class MediaController extends Controller
{
    /**
     * Store Media
     * @param MediaRequest $request
     * @return \Illuminate\Contracts\Routing\ResponseFactory|\Illuminate\Http\Response
     */
    public function store(MediaRequest $request)
    {
        $path = $request->file('file')->storePublicly("media/{$request->user()->id}", [
            'disk' => 'public',
        ]);

        return response([
            'url' => asset("storage/{$path}"),
        ]);
    }
}

```

## Request
```
<?php declare(strict_types=1);

namespace App\Http\Requests;
use Illuminate\Support\Facades\Gate;
use Illuminate\Foundation\Http\FormRequest;

class MediaRequest extends FormRequest
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
            'file' =>[
                'required',
                'mimes:png,jpg,jpeg,gif,bmp,webp',
                'max:30000'
            ]
        ];
    }
}

```