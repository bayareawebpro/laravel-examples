
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
        /** @var \Illuminate\Http\UploadedFile $file */
        $file = $request->file('file');

        $path = $file->storePublicly("media/{$request->user()->id}", [
            'disk' => 'public',
        ]);

        return response([
            'uploaded' => [
                'size' => $file->getSize(),
                'name' => $file->getClientOriginalName(),
                'mime' => $file->getMimeType(),
                'url' => asset("storage/{$path}"),
            ],
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
 
## Headers
 ```
//shortform
const headers = window.axios.defaults.headers.common

//longform
const headers = { 
    'X-Requested-With': 'XMLHttpRequest',
    'X-CSRF-TOKEN': document.getElementById('app').dataset.token,
}
```

## Component
 - `npm install dropzone`
 - `npm install axios`
 
```
<script>
    require('dropzone/dist/dropzone.css')
    import Dropzone from 'dropzone'
    Dropzone.autoDiscover = false
    export default {
        data: ()=>({
            errors: [],
        }),
        methods:{
            save(){
                this.errors = []
                this.$options.zone.processQueue()
            }
        },
        created() {
            this.$nextTick(()=>{
                this.$options.zone = new Dropzone(this.$refs.zone, {
                    headers: window.axios.defaults.headers.common,
                    acceptedFiles: 'image/*',
                    autoProcessQueue: false,
                    maxFilesize: 2, // MB
                    url: "/media/upload",
                    paramName: "file",
                    params:{
                        id: 1
                    },
                });
                this.$options.zone.on("addedfile", (file) =>{
                    console.log('addedfile', file)
                });
                this.$options.zone.on("removedfile", (file) =>{
                    console.log('removedfile', file)
                });
                this.$options.zone.on("success", (file, {uploaded}) =>{
                    console.log('success', file)
                    console.log('success', uploaded)
                });
                this.$options.zone.on("error", (file, {message}) =>{
                    this.errors.push(message)
                });
            })
        }
    }
</script>
<template>
    <div>
        <div v-for="error in errors" class="text-red-500">{{ error }}</div>
        <div ref="zone" class="dropzone"></div>
        <button @click="save">Save</button>
    </div>
</template>
```