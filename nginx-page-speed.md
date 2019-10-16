```
include forge-conf/site.com/server/*;

...add after forge includes:
include /home/forge/nginx-page-speed.conf;
```

```
# Media: images, icons, video, audio, HTC
location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
  expires 1M;
  access_log off;
  add_header Cache-Control "public";
}

# CSS and Javascript
location ~* \.(?:css|js|json)$ {
  expires 1M;
  access_log off;
  add_header Cache-Control "public";
}

# Compression
# Enable Gzip compressed.
gzip on;

# Enable compression both for HTTP/1.0 and HTTP/1.1.
gzip_http_version  1.1;

# Compression level (1-9).
# 5 is a perfect compromise between size and cpu usage, offering about
# 75% reduction for most ascii files (almost identical to level 9).
gzip_comp_level    5;

# Don't compress anything that's already small and unlikely to shrink much
# if at all (the default is 20 bytes, which is bad as that usually leads to
# larger files after gzipping).
gzip_min_length    256;

# Compress data even for clients that are connecting to us via proxies,
# identified by the "Via" header (required for CloudFront).
gzip_proxied       any;

# Tell proxies to cache both the gzipped and regular version of a resource
# whenever the client's Accept-Encoding capabilities header varies;
# Avoids the issue where a non-gzip capable client (which is extremely rare
# today) would display gibberish if their proxy gave them the gzipped version.
gzip_vary          on;

# Compress all output labeled with one of the following MIME-types.
gzip_types
#text/html
text/css
text/plain
text/xml
text/x-component
text/javascript
application/x-javascript
application/javascript
application/json
application/manifest+json
application/xml
application/xhtml+xml
application/rss+xml
application/atom+xml
application/vnd.ms-fontobject
application/x-font-ttf
application/x-font-opentype
application/x-font-truetype
image/svg+xml
image/x-icon
image/vnd.microsoft.icon
font/ttf
font/eot
font/otf
font/opentype;

```