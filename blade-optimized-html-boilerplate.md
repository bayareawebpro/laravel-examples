# Optimized Application HTML Shell

```blade
<!DOCTYPE html>
<html lang="en-US" prefix="og:http://ogp.me/ns#">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>

    <meta name="csrf-token" content="{{ csrf_token() }}">

    <link rel="preconnect" href="{{ config('app.url') }}">
    <link rel="preload" href="{{ asset(mix('css/app.css')) }}" as="style">
    <link rel="preload" href="{{ asset(mix('js/manifest.js')) }}" as="script">
    <link rel="preload" href="{{ asset(mix('js/vendor.js')) }}" as="script">
    <link rel="preload" href="{{ asset(mix('js/app.js')) }}" as="script">

    <link rel="preconnect" href="{{ config('filesystems.disks.spaces.edge') }}" crossorigin>
    <link rel="preconnect" href="https://www.google-analytics.com" crossorigin>
    <link rel="preconnect" href="https://www.googletagmanager.com" crossorigin>
    <link rel="preconnect" href="https://maps.googleapis.com" crossorigin>
    <link rel="preconnect" href="https://maps.gstatic.com" crossorigin>
    <link rel="preconnect" href="https://i3.ytimg.com" crossorigin>
 
    <!--Meta Title-->
    <title>{{ config('seo.meta_title', config('seo.meta_site_name')) }}</title>

    <!--OG Meta Title-->
    <meta property="og:title" content="{{ config('seo.meta_title', config('seo.meta_site_name')) }}"/>

    <!--Meta Description-->
    <meta name="description" content="{{ config('seo.meta_description') }}">

    <!--Meta OG Description-->
    <meta property="og:description" content="{{ config('seo.meta_description') }}">

    <!--Meta OG Site Name-->
    <meta property="og:site_name" content="{{ config('seo.meta_site_name') }}"/>

    <!--Meta OG Image-->
    <meta property="og:image" content="{{ asset(config('seo.meta_image')) }}"/>

    <!--Meta OG URL-->
    <meta property="og:url" content="{{ request()->url() }}">

    <!--Meta Canonical Url-->
    <link rel="canonical" href="{{ request()->url() }}">

    <!--Meta OG Type-->
    <meta property="og:type" content="website"/>

    <!--Meta Robots -->
    <meta name="robots" content="{{ config('seo.meta_robots') }}"/>

    <!--Mobile header color for Chrome, Firefox OS and Opera-->
    <meta name="theme-color" content="#ffffff"/>

    <!--Mobile header color Windows Phone-->
    <meta name="msapplication-navbutton-color" content="#2196f3"/>

    <!--Mobile header color for iOS Safari (supports black, black-translucent or default)-->
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent"/>

    <!--FavIcons-->
    <link rel="shortcut icon" type="image/x-icon" href="{{ asset('favicon.ico') }}">
    <link rel="icon" type="image/x-icon" href="{{ asset('favicon.ico') }}">

    <!--WebApp Manifest-->
    <link rel="manifest" href="{{ asset('manifest.json') }}">

    <!--Fallback PolyFills-->
    @include('scripts.polyfills')

    <!--Required External Vendor Async-->
    @include('scripts.google.places')

    <!--Application-->
    <link rel="stylesheet" href="{{ asset(mix('css/app.css')) }}"/>
    <script src="{{ asset(mix('js/manifest.js')) }}" defer></script>
    <script src="{{ asset(mix('js/vendor.js')) }}" defer></script>
    <script src="{{ asset(mix('js/app.js')) }}" defer></script>

    <!--Analytics External Vendors Async-->
    @include('scripts.google.analytics')

    <!--Stack Dynamic Includes-->
    @stack('head')
</head>
<body>
    <div id="app"></div>
    <noscript>
        <!--Fallback Content-->
    </noscript>
</body>
</html>
```