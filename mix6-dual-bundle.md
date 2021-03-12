# Laravel Mix Dual Bundle (v4)

## Output Directory Structure
```
public
  backend
    css
    fonts
    js
    mix-manifest.json
  frontend
    css
    fonts
    js
    mix-manifest.json
```

## Resource Directory Structure
```
resources
  backend
    css
    fonts
    js
  frontend
    css
    fonts
    js
```

## Config Files
```
tailwind-back.config.js
tailwind-front.config.js
webpack-back.mix.js
webpack-front.mix.js
```

## Tailwind Config (one per bundle)

Create a Tailwind config file for each bundle, modifying the PurgeCSS paths to match the subdirectory of each bundle.

```javascript
{
    ...
    purge: {
        content: [
            './resources/backend/**/*.blade.php',
            './resources/backend/**/*.js',
            './resources/backend/**/*.vue',
        ],
    },
}
```

## Mix Config (one per bundle)

Create a mix file for each bundle, modifying the paths to match the subdirectory of each bundle.

```javascript
cconst mix = require('laravel-mix');
const WebpackRequireFrom = require("webpack-require-from");
mix
    .setResourceRoot('/backend/') //Resource Root for Fonts & Images
    .setPublicPath('public/backend') //Mix Output Path
    
    .js('resources/backend/js/app.js', 'backend/js')
    .postCss('resources/backend/css/app.css', 'backend/css', [
        require("tailwindcss")('tailwind-back.config.js'),
        require('autoprefixer'),
    ])
    .webpackConfig({
        output: {
            publicPath: "/backend/", //Dynamic Imports Base Path
            chunkFilename: "js/[name].[chunkhash].js", //Code Splitting Chunk
        },
        plugins:[
            new WebpackRequireFrom //Dynamic Imports Plugin
        ],
        optimization: {
            providedExports: false,
            sideEffects: false,
            usedExports: false
        },
    })
    .vue({
        version: 3,
        extractStyles: false,
        globalStyles: false
    })
    .extract()
    .version();
```

## View Config Paths 

Modify `config/view.php` to specify the new view paths.

```php
'paths' => [
  resource_path('frontend/views'),
  resource_path('backend/views'),
  resource_path('views'), //vendor
],
```

## Customize NPM Scripts
```json
"scripts": {
    "back:dev": "mix --mix-config=webpack-back.mix.js",
    "back:watch": "mix watch --mix-config=webpack-back.mix.js",
    "back:watch-poll": "mix watch -- --watch-options-poll=1000 --mix-config=webpack-back.mix.js",
    "back:hot": "mix watch --hot --mix-config=webpack-back.mix.js",
    "back:prod": "mix --production --mix-config=webpack-back.mix.js",
    "front:dev": "mix --mix-config=webpack-front.mix.js",
    "front:watch": "mix watch --mix-config=webpack-front.mix.js",
    "front:watch-poll": "mix watch -- --watch-options-poll=1000 --mix-config=webpack-front.mix.js",
    "front:hot": "mix watch --hot --mix-config=webpack-front.mix.js",
    "front:prod": "mix --production --mix-config=webpack-front.mix.js"
},
```

## Include Assets

Use the mix helper second parameter to specify the bundle path.

```html
<script type="text/javascript" src="{{ asset(mix('js/manifest.js', 'backend')) }}" defer></script>
<script type="text/javascript" src="{{ asset(mix('js/vendor.js', 'backend')) }}" defer></script>
<script type="text/javascript" src="{{ asset(mix('js/app.js', 'backend')) }}" defer></script>
<link href="{{ asset(mix('css/app.css','backend')) }}" rel="stylesheet">
```

## Blade View Namespaces

Add hint paths for views and components.

```php
View::addNamespace('backend', resource_path('backend/views')); // admin::layout
Blade::componentNamespace('App\View\Backend', 'backend'); // <x-admin::alert/>

View::addNamespace('frontend', resource_path('frontend/views')); // frontend::layout
Blade::componentNamespace('App\View\Frontend', 'frontend');  // <x-frontend::alert/>
```
