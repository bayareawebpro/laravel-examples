## Dual Usage

```
<a class="{{ isActive('/home', 'nav-link-active', 'nav-link') }}">
    <!-- // -->
</a>
```

```
@if(isActive('resources.*', true, false))
    <!-- // -->
@endif
```

```
{{ isActiveUrl(['/', 'home']) }}
```

## Helper Function
```
/**
 * Is Active Url?
 * @param string|array $patterns
 * @param string $returnValue
 * @param string $fallbackValue
 * @return boolean
 */
function isActiveUrl($patterns, $returnValue = 'active', $fallbackValue = null)
{
    $isActive = (request()->is($patterns) || request()->routeIs($patterns));
    return $isActive ? $class : $fallback;
}
```