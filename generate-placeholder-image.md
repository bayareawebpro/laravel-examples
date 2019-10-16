```
<?php
//Intervention Image Package:
//http://image.intervention.io
Route::get('generate', function(){

    $text = 'image/jpg';
    $width = 1600;
    $height = 1000;
    $centerH = $width/2;
    $centerV = $height/2;
    $cropW = 640;
    $cropH = 480;

    $svg = base_path('node_modules/@fortawesome/fontawesome-free/svgs/regular/file.svg');
    $svg = \Intervention\Image\Facades\Image::make($svg);
    $svg->opacity(15);

    $img = \Intervention\Image\Facades\Image::canvas($width, $height, '#ff0000');
    $img->fill('#f3f3f3', 0, 0);
    $img->ellipse($height/1.2, $height/1.2, $centerH, $centerV, function ($draw) {
        $draw->background('#fff');
    });

    $img->insert($svg, 'center');
    $img->text($text, $centerH, $centerV, function($font) {
        $font->file(base_path('fonts/MicroSquare Bold.ttf'));
        $font->size(150);
        $font->color('#000');
        $font->align('center');
        $font->valign('middle');
    });

    $img->heighten($cropH);
    $img->crop($cropW,$cropH);
    return $img->response('jpg', 90);
});
```