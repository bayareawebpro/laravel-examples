> https://github.com/codingexplained/complete-guide-to-elasticsearch

## Scout Endpoint
```
SCOUT_ELASTIC_HOST=https://user_name:password@x.x.x.x:443
```

#### MySql Optimizations
```
query_cache_type=1
query_cache_size = 10M
query_cache_limit=256k
```

## NGINX Config
```
location / {
    proxy_pass http://localhost:9200; //ElasticSearch Port
    proxy_http_version 1.1;
    proxy_cache_bypass $http_upgrade;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header Connection "Keep-Alive";
    proxy_set_header Proxy-Connection "Keep-Alive";

    satisfy all;
    allow 97.84.89.251; //App IP
    allow 127.0.0.1;    //Local IP
    deny all;

    auth_basic           "UNAUTHORIZED";
    auth_basic_user_file /home/forge/htpasswd;
}
```

## Fast Bulk Import
- https://github.com/moshe/elasticsearch_loader

```
 ~/Library/Python/2.7/bin/elasticsearch_loader 
--index locations 
--type locations 
--id-field id 
csv /Users/you/Projects/osm-app/database/planet-latest_geonames.csv
```

## Backup & Restore
- https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html

#### Index Disk Usage
```
GET /_cat/indices?v
```

#### Delete Index
```
DELETE /my_index
```

#### Create Snapshot Repository
```
PUT /_snapshot/locations
{
   "type": "fs",
   "settings": {
       "compress" : true,
       "location": "/Users/you/Data/ElasticSearchSnapshots"
   }
}
```

#### Create Snapshot
```
PUT /_snapshot/locations/snapshot_1?wait_for_completion=false
```

#### Snapshot Details
```
GET /_snapshot/locations/snapshot_1
```

#### Delete Snapshots
```
DELETE /_snapshot/locations
```

#### Restore
```
POST /_snapshot/locations/snapshot_1/_restore
```


#### Add Cluster
```
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_two": {
          "seeds": [
            "X.X.X.X:80"
          ],
          "transport.ping_schedule": "30s",
          "transport.compress": true
        }
      }
    }
  }
}
```

#### Remove Cluster
```
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_three": {
          "seeds": null 
        }
      }
    }
  }
}
```

#### Cluster Disk Allocation
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.threshold_enabled": false,
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%",
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.info.update.interval": "1m"
  }
}
```
