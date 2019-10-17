## Artisan Generate Paycheck Command

```
Artisan::command('generate:paycheck', function () {
    $this->info(exec("git log --after='last month' --date=short --pretty=format:'%h,%an,%ad,%s' > ~/Desktop/app-commit-log.csv"));
})->describe('Generate a new paycheck.');
```

```
git log --after='last month' --date=short --pretty=format:'%h,%an,%ad,%s' > ~/Desktop/app-commit-log.csv
```