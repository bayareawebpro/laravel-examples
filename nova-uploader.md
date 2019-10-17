Once images are optimized you don't really need multiple sizes (IMO) as they are rarely bigger then 100k.  If you want thumbnails, then you'll need to customize this implementation for your own use-case. 


## Model
```
<?php
namespace App;
use Illuminate\Database\Eloquent\Model;
class Media extends Model
{
    protected $fillable = array(
        'file',
        'mime',
        'width',
        'height',
        'size',
        'hash',
    );
    
    public function getPreviewAttribute(){
        //ToDo: Resolve Media CDN Origin or StoragePath.
    }
    
    public function getUrlAttribute(){
        //ToDo: Resolve Media CDN Edge or StoragePath.
    }
}
```

## Field
```
 Image::make('File', 'file')
    ->store(new FileUploader('spaces'))
    ->rules('mimes:jpg,jpeg,png,gif', 'max:5000')
    ->preview(function(){
        return $this->resource->getAttribute('preview');
    })
    ->thumbnail(function(){
        return $this->resource->getAttribute('preview');
    })
    ->delete(function ($request, $model){
        return Storage::disk('spaces')->delete("media/$model->file");
    })
    ->prunable($request->isMethod('delete'))
    ->deletable($request->isMethod('delete'))
    ->help('Upload or Replace: Filename & MimeType will be preserved.')
    ->maxWidth(640),
```

## Invokeable Class
```
<?php namespace App\Nova\Invokables;

use App\Media;

use Illuminate\Support\Str;
use Illuminate\Http\Request;
use Illuminate\Http\UploadedFile;

use Intervention\Image\Facades\Image;

use Spatie\ImageOptimizer\OptimizerChain;
use Spatie\ImageOptimizer\Optimizers\Svgo;
use Spatie\ImageOptimizer\Optimizers\Optipng;
use Spatie\ImageOptimizer\Optimizers\Pngquant;
use Spatie\ImageOptimizer\Optimizers\Gifsicle;
use Spatie\ImageOptimizer\Optimizers\Jpegoptim;

class FileUploader
{

    static $mimeTypes = array(
        'image/jpeg',
        'image/jpg',
        'image/png',
        'image/gif',
    );

    static $maxWidth = 1024;
    static $maxHeight = 768;

    protected
        $needsEncodingTo,
        $originalName,
        $sanitizedName,
        $uploadedFile,
        $needsResize,
        $attributes,
        $extension,
        $directory,
        $tempPath,
        $filename,
        $mimeType,
        $height,
        $width,
        $image,
        $model,
        $disk,
        $hash;

    /**
     * FileUploader constructor.
     * @param string $disk
     * @param string $directory
     */
    public function __construct($disk = 'spaces', $directory = 'media')
    {
        $this->directory = $directory;
        $this->disk = $disk;
    }

    /**
     * @param Request|UploadedFile $instance
     * @param Media $model
     * @return array
     * @throws \Exception
     */
    public function __invoke($instance, Media $model)
    {
        $this->model = $model;
        $this->setFileInstance($instance);
        $this->verifyUpload();
        $this->determineEncoding();
        $this->resizeAndEncode();
        $this->optimize();
        $this->attributes = $this->makeAttributes();
        $this->uploadedFile->storePubliclyAs($this->directory, $this->attributes['file'], [
            'disk'=>  $this->disk,
            'visibility' => 'public'
        ]);
        return $this->attributes;
    }

    /**
     * Set File Instance.
     * @param $instance
     * @throws \Exception
     */
    protected function setFileInstance($instance){
        if($instance instanceof Request){
            $this->uploadedFile = $instance->file('file');
        }elseif($instance instanceof UploadedFile){
            $this->uploadedFile = $instance;
        }else{
            throw new \Exception("instanceof Request or UploadedFile expected.");
        }
        $this->originalName = $this->uploadedFile->getClientOriginalName();
        $this->extension = $this->uploadedFile->guessClientExtension();
        $this->mimeType = $this->uploadedFile->getMimeType();
        $this->tempPath = $this->uploadedFile->path();
        $this->sanitizedName = $this->makeFilename();
        $this->hash = hash_file('md5', $this->tempPath);;
    }

    /**
     * Make Model Attributes
     * @return mixed
     */
    protected function makeAttributes()
    {
        if (is_null($this->model->file)) {
            $this->model->file = $this->sanitizedName;
        }

        if (is_null($this->model->mime)) {
            $this->model->mime = $this->mimeType;
        }

        $this->model->hash = $this->hash;
        $this->model->width = $this->width;
        $this->model->height = $this->height;
        $this->model->size = app('files')->size($this->tempPath);

        return $this->model->only([
            'height',
            'width',
            'size',
            'mime',
            'file',
            'hash',
        ]);
    }

    /**
     * Make Sanitized Filename.
     * @return mixed|string
     */
    protected function makeFilename()
    {
        $filename = $this->getFilename($this->originalName);
        $filename = Str::limit($filename, 250, '');
        $filename = Str::slug($filename);
        $filename = $filename . ".{$this->getExtension($this->originalName)}";
        return $filename;
    }

    /**
     * Should Convert File Type?
     * @return void
     */
    protected function determineEncoding()
    {
        if (!is_null($this->model->mime)) {
            if ($this->mimeType !== $this->model->mime) {
                $this->needsEncodingTo = $this->model->mime;
            }
        }
    }

    /**
     * Perform Resize & Conversion Operations.
     * @return void
     */
    protected function resizeAndEncode()
    {
        ini_set('memory_limit', '256M');

        $this->image = Image::make($this->tempPath);

        $this->width = $this->image->width();
        $this->height = $this->image->height();

        $this->needsResize = ($this->width > static::$maxWidth || $this->height > static::$maxHeight);

        if ($this->needsResize) {
            $this->image->fit(static::$maxWidth, static::$maxHeight, function ($constraint) {
                /** @var $constraint \Intervention\Image\Constraint */
                $constraint->upsize();
            });
        }

        if ($this->needsEncodingTo) {
            $this->image->encode($this->needsEncodingTo, 75);
        }

        if ($this->needsEncodingTo || $this->needsResize) {
            $this->image->save($this->tempPath, 75);
        }
    }

    /**
     * Perform Optimization Operations.
     * @return void
     */
    protected function optimize()
    {
        try {
            $binaryPath = config('image.optimizer_binary_path');
            $optimizerChain = (new OptimizerChain())
                ->addOptimizer(
                    with(new Jpegoptim([
                        '--max75',
                        '--strip-all',
                        '--all-progressive',
                        '--quiet'
                    ]))
                    ->setBinaryPath($binaryPath)
                )
                ->addOptimizer(
                    with(new Optipng([
                        '-i0',
                        '-o3',
                        '-quiet',
                    ]))
                    ->setBinaryPath($binaryPath)
                )
                ->addOptimizer(
                    with(new Pngquant([
                        '--force',
                        '--skip-if-larger',
                        '--quality=75'
                    ]))
                    ->setBinaryPath($binaryPath)
                )
                //->addOptimizer(
                //    with(new Svgo([
                //        '--disable=cleanupIDs',
                //    ]))
                //    ->setBinaryPath($binaryPath)
                //)
                ->addOptimizer(
                    with(new Gifsicle([
                        '-b',
                        '-O3',
                    ]))
                    ->setBinaryPath($binaryPath)
                );
            $optimizerChain->useLogger(app('log'));
            $optimizerChain->optimize($this->tempPath, $this->tempPath);
        } catch (\Exception $e) {
            logger()->error($e->getMessage(), $e->getTrace());
        }
    }

    /**
     * Get the Extension from a string.
     * @param string $filename
     * @return mixed
     */
    protected function getExtension($filename)
    {
        return pathinfo($filename, PATHINFO_EXTENSION);
    }

    /**
     * Get the Filename from a string.
     * @param string $filename
     * @return mixed
     */
    protected function getFilename($filename)
    {
        return pathinfo($filename, PATHINFO_FILENAME);
    }

    /**
     * Verify the File's MimeType.
     * @throws \Exception
     */
    protected function verifyUpload()
    {
        if (!in_array($this->mimeType, $this::$mimeTypes)) {
            throw new \Exception("FileType not allowed.");
        }
        if (is_null($this->model->id)) {
            if ($this->model->where('hash', $this->hash)->orWhere('file', $this->sanitizedName)->exists()) {
                throw new \Exception("Duplicate File Detected, Upload Rejected.");
            }
        }
    }
}

```

## Unit Test
```
<?php
namespace Tests\Unit;
use Illuminate\Http\Request;
use Illuminate\Http\UploadedFile;
use Intervention\Image\Facades\Image;
use App\Nova\Invokables\FileUploader;
use Illuminate\Support\Facades\Storage;
use Tests\TestCase;
use App\Media;
class FileUploaderTest extends TestCase
{

    /**
     * File Uploader General Test
     * (covers usage of: Spatie Image Optimizer & Intervention Image)
     * @return void
     * @throws \Throwable
     */
    public function test_it_can_resize_convert_and_optimize()
    {
        Storage::fake('spaces');

        $file = UploadedFile::fake()->image('user proviDed bull$shit name.jpg', 1600, 1200);

        $request = Request::create('/', 'POST', [], [], array(
            'file' => $file
        ));

        $upload = new FileUploader('spaces');
        $result = $upload($request, new Media);

        $expectedAttributes = [
            'height',
            'width',
            'size',
            'mime',
            'file',
            'hash',
        ];
        foreach($expectedAttributes as $attribute){
            $this->assertNotEmpty($result[$attribute]);
        }

        $expectedName = 'user-provided-bullshit-name.jpg';

        $this->assertEquals($expectedName, $result['file']);

        // Assert the file was stored...
        Storage::disk('spaces')->assertExists("media/$expectedName");

        // Read the newly created image from the testing directory...
        $image = Image::make(storage_path("framework/testing/disks/spaces/media/$expectedName"));

        // Assert Attributes
        $this->assertEquals('image/jpeg', $image->mime());
        $this->assertLessThanOrEqual(FileUploader::$maxWidth, $image->width());
        $this->assertLessThanOrEqual(FileUploader::$maxHeight, $image->height());
    }



    /**
     * File Uploader Cross Encoding Test
     * (covers usage of: Image Encoding Related Class Methods)
     * @return void
     * @throws \Throwable
     */
    public function test_it_can_encode_to_existing_mime()
    {

        Storage::fake('spaces');

        $file = UploadedFile::fake()->image('user-provided-jpg-image.jpg', 1600, 1200);

        $upload = new FileUploader('spaces');

        $request = Request::create('/', 'POST', [], [], array(
            'file' => $file
        ));

        $existingFilename = 'existing-user-uploaded-png-image.png';
        $existingMime = 'image/png';

        //Specify the model has an existing file of a different mime type.
        $result = $upload($request, new Media(array(
            'file' => $existingFilename,
            'mime' => $existingMime
        )));

        $expectedAttributes = [
            'height',
            'width',
            'size',
            'mime',
            'file',
            'hash',
        ];
        foreach($expectedAttributes as $attribute){
            $this->assertNotEmpty($result[$attribute]);
        }

        $this->assertEquals($existingFilename, $result['file']);
        $this->assertEquals($existingMime, $result['mime']);

        // Assert the file was stored...
        Storage::disk('spaces')->assertExists("media/$existingFilename");

        $image = Image::make(storage_path("framework/testing/disks/spaces/media/$existingFilename"));

        $this->assertEquals($existingMime, $image->mime());
        $this->assertLessThanOrEqual(FileUploader::$maxWidth, $image->width());
        $this->assertLessThanOrEqual(FileUploader::$maxHeight, $image->height());
    }
}

```
