## Implementation

```
$this->app->singleton('blade.compiler', function () {
    return new HtmlMinifyCompiler($this->app['files'], $this->app['config']['view.compiled']);
});
$this->app['view']->getEngineResolver()->register('blade', function () {
    return new CompilerEngine($this->app['blade.compiler']);
});
```

## Compiler 
```
<?php
class HtmlMinifyCompiler extends BladeCompiler
{
    public function __construct($fileSystem, $cachePath)
    {
        parent::__construct($fileSystem, $cachePath);
        if(config('view.minify', true)) {
            $this->compilers[] = 'Minify';
        }
    }
    /**
     * Compress the HTML output before saving it
     * @param string $input the contents of the view file
     * @return string
     */
    protected function compileMinify($input)
    {
        // Use this comment to disable per-view file: <!-- skip.minification -->
        if ($this->shouldMinify($input)) {
            // Remove HTML comments, but not SSI
            $input = preg_replace('/<!--[^#](.*?)-->/s', '', $input);
            // The content inside these tags will be spared:
            $doNotCompressTags = ['script', 'pre', 'textarea'];
            $matches = [];
            foreach ($doNotCompressTags as $tag) {
                $regex = "!<{$tag}[^>]*?>.*?</{$tag}>!is";
                // It is assumed that this placeholder could not appear organically in your
                // output. If it can, you may have an XSS problem.
                $placeholder = "@@<'-placeholder-$tag'>@@";
                // Replace all the tags (including their content) with a placeholder, and keep their contents for later.
                $input = preg_replace_callback(
                    $regex,
                    function ($match) use ($tag, &$matches, $placeholder) {
                        $matches[$tag][] = $match[0];
                        return $placeholder;
                    },
                    $input
                );
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
        //Log Result if Configured.
        $this->logResult($input);
        return $input;
    }
    /**
     * Log Compiled Output
     * @param string $input
     * @return void
     */
    protected function logResult($input){
        if(config('view.log', true)){
            $templateName = basename($this->getPath());
            $message = "Compiling Blade Template: $templateName".PHP_EOL;
            $message .= $input;
            app('log')->info($message);
        }
    }
    /**
     * Determine if the blade should be minified.
     * @param string $value
     * @return bool
     */
    protected function shouldMinify($value)
    {
        return !$this->containsBadHtml($value) && !$this->containsBadComments($value);
    }
    /**
     * Does the code contain bad html?
     * @param string $value
     * @return bool
     */
    protected function containsBadHtml($value)
    {
        return preg_match('/<(code|pre|textarea)/', $value) ||
            preg_match('/<script[^\??>]*>[^<\/script>]/', $value) ||
            preg_match('/<!--[\s]+skip\.minification[\s]+-->/', $value) ||
            preg_match('/value=("|\')(.*)([ ]{2,})(.*)("|\')/', $value);
    }
    /**
     * Does the code contain bad comments?
     * @param string $value
     * @return bool
     */
    protected function containsBadComments($value)
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