```
#!/usr/bin/env bash
echo "Optimizing Remote Database..."

ssh -C forge@138.68.50.99 "cd ~/site.com/current && php artisan telescope:clear"

echo "Synchronizing Remote Database..."
ssh -C forge@X.X.X.X "mysqldump --default-character-set=utf8mb4 staging" | /Applications/MAMP/Library/bin/mysql staging
echo "Staging Synchronized Successfully."

ssh -C forge@X.X.X.X "mysqldump --default-character-set=utf8mb4 production" | /Applications/MAMP/Library/bin/mysql production
echo "Production Synchronized Successfully."
```

```
#!/usr/bin/env bash
echo "Overwriting Staging & Production is Forbidden!  Nice Try Gangsta. UnComment if you dare!"

#php artisan telescope:clear
#/Applications/MAMP/Library/bin/mysqldump staging | ssh -C forge@X.X.X.X "mysql staging"
#/Applications/MAMP/Library/bin/mysqldump production | ssh -C forge@X.X.X.X "mysql production"
```