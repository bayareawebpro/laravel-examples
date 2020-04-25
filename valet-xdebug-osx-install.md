# OSX Laravel Valet xDebug


### Verify Symlink

Verify the Symlink for brew's php version.  If it's broken, delete it.

```shell script
/usr/local/Cellar/php/7.4.0/pecl
```

### Install Extension

```shell script
pecl install xdebug
```

### Update PHP.ini
```
;XDebug
zend_extension="/usr/local/Cellar/php/7.4.0/pecl/20190902/xdebug.so"
;xdebug.profiler_output_dir="/Users/me/xdebugtmp/"
;xdebug.remote_autostart=1
;xdebug.profiler_enable=1
xdebug.remote_enable=1
xdebug.remote_port=9000
```
