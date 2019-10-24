# Helper: Bytes for Humans 

```php
<?php
if (!function_exists('bytesForHumans')) {
	/**
	 * Bytes to Human Readable
	 * @param $bytes int
	 * @param $formatted bool
	 * @return mixed
	 */
	function bytesForHumans($bytes, $formatted = true){
		$units = ['B', 'KiB', 'MiB', 'GiB', 'TiB', 'PiB'];
		for ($i = 0; $bytes > 1024; $i++) {
			$bytes /= 1024;
		}
		return round($bytes, 2) . ' ' . ($formatted ? $units[$i] : null);
	}
}
```