## Markdown Renderer

```php
$rendered = Markdown::parse('my/page');
```

```php
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\File;
use League\CommonMark\GithubFlavoredMarkdownConverter;

class Markdown{

    public static function parse(string $path)
    {
        $path = resource_path("markdown/$path.md");

        if(File::exists($path)){
            return with(new GithubFlavoredMarkdownConverter([
                //'html_input' => 'strip',
                //'allow_unsafe_links' => false,
            ]))->convertToHtml(File::get($path));
        }
    }
}
```