# Synchronize Remote Databases

## Sync Down
```shell script
#!/usr/bin/env bash

echo "Optimizing Remote Database..."
ssh -C forge@X.X.X.X "cd ~/site.com/current && php artisan telescope:clear"

echo "Synchronizing Local with Remote..."
ssh -C forge@X.X.X.X "mysqldump --default-character-set=utf8mb4 staging" | mysql staging
echo "Staging Synchronized to Local Successfully."

ssh -C forge@X.X.X.X "mysqldump --default-character-set=utf8mb4 production" | mysql production
echo "Production Synchronized to Local Successfully."
```

## Sync Up
```shell script
#!/usr/bin/env bash

echo "Optimizing Local Database..."
php artisan telescope:clear

echo "Synchronizing Remote with Local..."
mysqldump staging | ssh -C forge@X.X.X.X "mysql staging"
echo "Staging Synchronized to Remote Successfully."

echo "Writing to Production Remote is Forbidden!  Nice Try Gangsta. UnComment if you dare!"
#mysqldump production | ssh -C forge@X.X.X.X "mysql production"
#echo "Production Synchronized to Remote Successfully."

```
