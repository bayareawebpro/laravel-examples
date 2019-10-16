Laravel Model & Index
https://gist.github.com/bayareawebpro/ab53ccd81a4855b5267aead158c21670
https://gist.github.com/bayareawebpro/bb8c3a03718f6dc70a8ed569e4465f7e


## Insure All Fields All Filled or Null to prevent indexing errors.
```
/**
 * Get the indexable data array for the model.
 * @return array
 */
public function toSearchableArray()
{
    return [
        'field' => $this->field ?? null
    ];
}
```


## Bulk Import
https://github.com/moshe/elasticsearch_loader

```
 ~/Library/Python/2.7/bin/elasticsearch_loader 
--index locations 
--type locations 
--id-field id 
csv /Users/builder/Projects/osm-app/database/planet-latest_geonames.csv
```

## ElasticSearch Backup & Restore
https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html


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
## Scout Endpoint
```
SCOUT_ELASTIC_HOST=https://elasticsearch:tyr5MY.kvB1EYPt@X.X.X.X:443
```

## NGINX Config
```
satisfy all;    
allow x.x.x.x;
allow 127.0.0.1;
deny  all;
auth_basic           "UNAUTHORIZED";
auth_basic_user_file /home/forge/htpasswd;
user: elasticsearch 
pass: tyr5MY.kvB1EYPt
```

#### Index Disk Usage
```
GET /_cat/indices?v
```

#### Create Snapshot Repository
```
PUT /_snapshot/locations
{
   "type": "fs",
   "settings": {
       "compress" : true,
       "location": "/Users/builder/Data/ElasticSearchSnapshots"
   }
}
```

#### Delete Index
```
DELETE /my_index
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

#### MySql Forge Recommended Database Optimizations
```
query_cache_type=1
query_cache_size = 10M
query_cache_limit=256k
```
