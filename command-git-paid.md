## Artisan GIT PAID

```
Artisan::command('git:paid', function () {
    $this->info(exec("git log --after='last month' --date=short --pretty=format:'%h,%an,%ad,%s' > ~/Desktop/app-commit-log.csv"));
})->describe('Generate a new commit log for last month.');
```

```
git log --after='last month' --date=short --pretty=format:'%h,%an,%ad,%s' > ~/Desktop/app-commit-log.csv
```