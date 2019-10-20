
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
 - `npm install vue`
 - `npm install dropzone`
 
#### Usage
 ```vue
<dropzone route="/upload"></dropzone>
```

#### Custom Preview
 ```vue
<dropzone route="/upload">
    <!-- Custom Preview -->
    <div class="dz-preview dz-file-preview">
        <div class="dz-details">
            <div class="dz-size" data-dz-size></div>
            <div class="dz-filename">
                <span data-dz-name></span>
            </div>
        </div>
        <div class="dz-image">
            <img data-dz-thumbnail/>
        </div>
        <div class="dz-progress"><span class="dz-upload" data-dz-uploadprogress></span></div>
        <div class="dz-success-mark"><span>✔</span></div>
        <div class="dz-error-mark"><span>✘</span></div>
        <div class="dz-error-message"><span data-dz-errormessage></span></div>
    </div>
</dropzone>
```

#### Implementation
```vue
<script>
    import Dropzone from 'dropzone'
    Dropzone.autoDiscover = false
    export default {
        props: ['route'],
        data: () => ({errors: []}),
        computed: {
            previewTemplate() {
                if (this.$slots.default && this.$slots.default[0].elm) {
                    return this.$slots.default[0].elm.parentNode.innerHTML
                }
            }
        },
        methods: {
            save() {
                this.errors = []
                this.$options.zone.processQueue()
            },
            showUploadedFile() {
                let mockFile = {
                    id: 123,
                    size: 123,
                    name: "image.jpg",
                    type: 'image/jpeg'
                }
                // Call the default addedfile event handler
                this.$options.zone.emit("addedfile", mockFile);

                // And optionally show the thumbnail of the file:
                if (mockFile.url) {
                    this.$options.zone.emit("thumbnail", mockFile, mockFile.url);
                    //this.$options.zone.createThumbnailFromUrl(mockFile, mockFile.url);
                }

                // Make sure that there is no progress bar, etc...
                this.$options.zone.emit("complete", mockFile);

                // If you use the maxFiles option, make sure you adjust it
                if (this.$options.zone.options.maxFiles > 1) {
                    this.$options.zone.options.maxFiles = this.$options.zone.options.maxFiles - 1
                }
            }
        },
        created() {
            this.$nextTick(() => {
                this.$options.zone = new Dropzone(this.$refs.zone, {
                    //confirm: (question, accepted, rejected = null) => {},
                    headers: window.axios.defaults.headers.common,
                    acceptedFiles: 'image/*',
                    ignoreHiddenFiles: false,
                    autoProcessQueue: false,
                    addRemoveLinks: true,
                    thumbnailHeight: 300,
                    thumbnailWidth: 300,
                    maxFilesize: 2, // MB
                    maxFiles: 10,
                    url: this.route,
                    paramName: "file",
                    params: {
                        id: 1
                    },
                })
                // If the component has a preview template, use it.
                if (this.previewTemplate) {
                    this.$options.zone.options.previewTemplate = this.previewTemplate
                }
                // If the file is a preview we need to add it to the file array.
                this.$options.zone.on("addedfile", (file) => {
                    console.log('addedfile', file, this.$options.zone.files)
                    if (this.$options.zone.files.indexOf(file) < 0) {
                        this.$options.zone.files.push(file)
                    }
                })
                // allow queue to continue looping all the files
                this.$options.zone.on("success", (file, {uploaded}) => {
                    this.$options.zone.options.autoProcessQueue = true;
                    console.info('success', uploaded)
                })
                // disable auto processing the queue
                this.$options.zone.on("queuecomplete", () => {
                    this.$options.zone.options.autoProcessQueue = false;
                })
                // delete a file from the server?
                this.$options.zone.on("removedfile", (file) => {
                    console.log('removedfile', file)
                })
                // sync errors
                this.$options.zone.on("error", (file, {message}) => {
                    this.errors.push(message)
                    console.error('error', message)
                })
            })
        }
    }
</script>
<style lang="sass">
    @import "~dropzone/dist/dropzone.css"
</style>
<template>
    <div>
        <div v-for="error in errors">{{ error }}</div>
        <div ref="zone" class="dropzone"></div>

        <button @click="save">Save</button>
        <button @click="showUploadedFile">showUploadedFile</button>

        <div class="hidden">
            <slot>
                <div class="dz-preview dz-file-preview">
                    <div class="dz-details">
                        <div class="dz-size" data-dz-size></div>
                        <div class="dz-filename">
                            <span data-dz-name></span>
                        </div>
                    </div>
                    <div class="dz-image">
                        <img data-dz-thumbnail/>
                    </div>
                    <div class="dz-progress"><span class="dz-upload" data-dz-uploadprogress></span></div>
                    <div class="dz-success-mark"><span>✔</span></div>
                    <div class="dz-error-mark"><span>✘</span></div>
                    <div class="dz-error-message"><span data-dz-errormessage></span></div>
                </div>
            </slot>
        </div>
    </div>
</template>
```