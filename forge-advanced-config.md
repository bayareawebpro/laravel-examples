# Laravel Forge Advanced NGINX Config

## Rate Limiting

Declare a rate limiting zone at the top of the forge config above the server blocks.

```nginx
limit_req_zone $binary_remote_addr zone=production:10m rate=10r/s;
```

#### Apply Rate Limiting to PHP-FPM
```nginx
location ~ \.php$ {
    # Rate Limited
    limit_req zone=production burst=30 delay=15;
    # ...
}
```

## Page Speed

Apply the page speed config & any custom SEO redirects to the primary server configs via the UI.

```nginx
server {
    # Forge SSL...

    # Include Page Speed Directives
    include /home/forge/nginx-page-speed.conf;
    include /home/forge/nginx-redirects-staging.conf;
    
    # Try Files & Index...
}
```

#### Page Speed Config

Upload to `/home/forge/nginx-page-speed.conf`

```nginx
# Media & Misc
location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
  expires 1M;
  access_log off;
  add_header Cache-Control "public";
}

# CSS, Javascript & JSON files.
location ~* \.(?:css|js|json)$ {
  expires 1M;
  access_log off;
  add_header Cache-Control "public";
}

# GZIP Settings
gzip on;
gzip_http_version  1.1;
gzip_comp_level    5;
gzip_min_length    256;
gzip_proxied       any;
gzip_vary          on;

# GZIP FileTypes
gzip_types
# duplicate: text/html
text/css
text/xml
text/plain
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



## A+ SSL Wide Compatibility Including Subdomains

Update each server block to include the custom SSL Conf. The alias 
redirects are not editable via the forge UI.
```
/etc/nginx/forge-conf/site.com/before/ssl_redirect.conf
/etc/nginx/forge-conf/www.site.com/before/ssl_redirect.conf
/etc/nginx/forge-conf/staging.site.com/before/ssl_redirect.conf
```

#### Edit Config
```shell script
sudo nano /etc/nginx/forge-conf/site.com/before/ssl_redirect.conf
```

#### Test Config
```shell script
sudo nginx -t
```

#### Restart Server
```shell script
sudo service nginx restart
```

#### Example Config Edits:
```nginx
server {
    # Disable Forge Directives
    # ssl_protocols TLSv1.2;
    # ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
    # ssl_prefer_server_ciphers on;

    # Include Custom Directives
    include /home/forge/nginx-ssl.conf;
}
```

#### NGINX SSL Config

Upload to `/home/forge/nginx-ssl.conf`

> Notice:  These directives are less secure (IE Support) than the forge defaults and should not be used for any application that stores sensitive data.  Wide compatibility comes with tradeoffs.  That being said, choose the cyphers and protocols that best suite your application and customer needs.

```nginx
# Add Cyphers
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
#ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; #Use v1.3 at your own risk!
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA;
ssl_prefer_server_ciphers on;

# Add Session Timeout, Cache & Tickets.
ssl_session_timeout 1d;
ssl_session_cache shared:MozSSL:10m;
ssl_session_tickets off;

# OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;

# HSTS (ngx_http_headers_module is required) (63072000 seconds)
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
```
