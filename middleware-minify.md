

HTML Minifier Middleware
```
<?php declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use App\Services\HtmlMinifier;
use App\Services\ResponseFilter;

class MinifyOutput
{
    /**
     * Handle an incoming request.
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        if($request->isMethod('GET')){
            /** Minify the output if enabled. */
            if(config('response.minify.enabled', false) === true){
                $response->setContent(HtmlMinifier::minify($response->getContent()));
            }
        }
        return $response;
    }
}

```

HTML Minifier
```
<?php declare(strict_types=1);

namespace App\Services;

class HtmlMinifier
{
    /**
     * Compress the HTML output before saving it
     * @param string $input the contents of the view file
     * @return string
     */
    public static function minify($input)
    {
        try{
            // Use this comment to disable per-view file: <!-- skip.minification -->
            if (static::shouldMinify($input)) {

                // Remove HTML comments, but not SSI
                $input = preg_replace('/<!--[^#](.*?)-->/s', '', $input);

                // The content inside these tags will be spared:
                $doNotCompressTags = ['script', 'textarea', 'pre', 'code'];
                $matches = [];

                foreach ($doNotCompressTags as $tag) {
                    $regex = "!<{$tag}[^>]*?>.*?</{$tag}>!is";

                    // It is assumed that this placeholder could not appear organically in your
                    // output. If it can, you may have an XSS problem.
                    $placeholder = "@@<'-placeholder-$tag'>@@";

                    // Replace all the tags (including their content) with a placeholder, and keep their contents for later.
                    $input = preg_replace_callback($regex, function ($match) use ($tag, &$matches, $placeholder) {
                        $matches[$tag][] = $match[0];
                        return $placeholder;
                    }, $input);
                }

                // Remove whitespace (spaces, newlines and tabs)
                $input = trim(preg_replace('/[ \n\t]+/m', ' ', $input));

                // Iterate the blocks we replaced with placeholders beforehand, and replace the placeholders
                // with the original content.
                foreach ($matches as $tag => $blocks) {
                    $placeholder = "@@<'-placeholder-$tag'>@@";
                    $placeholderLength = strlen($placeholder);
                    $position = 0;
                    foreach ($blocks as $block) {
                        $position = strpos($input, $placeholder, $position);
                        if ($position === false) {
                            throw new \RuntimeException("Found too many placeholders of type $tag in input string");
                        }
                        $input = substr_replace($input, $block, $position, $placeholderLength);
                    }
                }

                // Collapse Spaces between Tags from Final Output.
                $input = preg_replace('/\> \</', '><', $input);

            } else {
                // Where skip minification tags are used let's remove them from markdown or blade.
                $input = preg_replace("/<!--[\s]+skip\.minification[\s]+-->/", '', $input);
            }
        }catch (\Exception $e){
            logger()->critical($e->getMessage(), $e->getTrace());
        }

        return $input;
    }

    /**
     * Determine if the blade should be minified.
     * @param string $value
     * @return bool
     */
    protected static function shouldMinify($value)
    {
        return !static::containsBadHtml($value) && !static::containsBadComments($value);
    }

    /**
     * Does the code contain bad html?
     * @param string $value
     * @return bool
     */
    protected static function containsBadHtml($value)
    {
        return (
            preg_match('/<!--[\s]+skip\.minification[\s]+-->/', $value) ||
            preg_match('/value=("|\')(.*)([ ]{2,})(.*)("|\')/', $value)
        );
    }
    /**
     * Does the code contain bad comments?
     * @param string $value
     * @return bool
     */
    protected static function containsBadComments($value)
    {
        foreach (token_get_all($value) as $token) {
            if (!is_array($token) || !isset($token[0]) || $token[0] !== T_COMMENT) {
                continue;
            }
            if (substr($token[1], 0, 2) === '//') {
                return true;
            }
        }
        return false;
    }
}

```