## Usage

```
<a class="{{ isActive('resources.*', 'nav-link-active', 'nav-link') }}">
    <!-- // -->
</a>
```

```
@if(isActive('resources.*', true, false))
    <!-- // -->
@endif
```

## Helper Function
```
/**
 * Is Active Url?
 * @param string $slug
 * @param string $class
 * @param string $fallback
 * @return mixed
 */
function isActiveUrl($slug, $class = 'active', $fallback = null)
{
    $isActive = request()->is($slug);
    return $isActive ? $class : $fallback;
}
```