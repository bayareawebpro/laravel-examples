# GitHub Actions Workflow for Laravel

### Add Badge
```
![](https://github.com/my-account/my-repo/workflows/my-workflow-name/badge.svg)
```

---

## Package Workflow

Add to `.github/workflows/ci.yaml`

```
name: tests
on: [push, pull_request]
jobs:
  CI:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1
      - name: Install Composer Dependencies
        run: composer install --no-ansi --no-interaction --no-scripts --no-suggest --no-progress --prefer-dist --optimize-autoloader
      - name: Lint
        run: php ./vendor/bin/phpstan analyse
      - name: Testsuite
        run: php ./vendor/bin/phpunit
```

## Full-stack APP Workflow

Add to `.github/workflows/ci.yaml`

```
name: tests
on:
  push:
    paths-ignore:
      - 'README.md'
      - 'LICENSE.md'
      - 'docs'
  pull_request:
    paths-ignore:
      - 'README.md'
      - 'LICENSE.md'
      - 'docs'
    branches:
      - master
jobs:
  CI:
    runs-on: ubuntu-latest
    # container:
    #   image: your/laravel-docker
    steps:
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1

      - name: Configure Environment
        run: |
          cp .env.ci .env

      - name: Cache Composer
        uses: actions/cache@v1
        with:
          path: vendor
          key: ${{ runner.OS }}-build-${{ hashFiles('**/composer.lock') }}

      - name: Composer Dependencies
        run: |
          composer install --no-ansi --no-interaction --no-scripts --no-suggest --no-progress --prefer-dist --optimize-autoloader
          php artisan key:generate

      - name: Create Sqlite Database
        run: touch database/testing.sqlite

      - name: Run PHP Analysis
        run: ./vendor/bin/phpstan analyse

      - name: Run PHP Tests
        run: php ./vendor/bin/phpunit

      - name: Cache NPM
        uses: actions/cache@v1
        with:
          path: node_modules
          key: ${{ runner.OS }}-build-${{ hashFiles('**/package-lock.json') }}

      - name: Install Node
        uses: actions/setup-node@v1
        with:
          node-version: 12.x

      - name: Install Assets
        run: npm install --quiet --no-optional --prefer-offline --no-audit

      - name: Compile Assets
        run: npm run production

      - name: Install Chrome Driver
        run: |
          php artisan dusk:chrome-driver `/opt/google/chrome/chrome --version | cut -d " " -f3 | cut -d "." -f1`

      - name: Start Chrome Driver
        run: |
          ./vendor/laravel/dusk/bin/chromedriver-linux &

      - name: Serve Application
        run: php artisan serve &

      - name: Run Feature Tests
        run: php artisan dusk --stop-on-failure

      - name: Store Log Artifacts
        uses: actions/upload-artifact@v1
        if: failure()
        with:
          name: Logs
          path: ./storage/logs

      - name: Store Screenshot Artifacts
        uses: actions/upload-artifact@v1
        if: failure()
        with:
          name: Screenshots
          path: ./tests/Browser/screenshots
```


### Dusk Required Options

- `APP_URL=http://127.0.0.1:8000`
- `SANCTUM_DOMAIN=127.0.0.1`


- Constant: `ChromeOptions::CAPABILITY`
- Disable: `--disable-gpu`

```php
/**
 * Create the RemoteWebDriver instance.
 * @return \Facebook\WebDriver\Remote\RemoteWebDriver
 */
protected function driver()
{
    $options = (new ChromeOptions)->addArguments([
        //'--disable-gpu', Chrome doesn't load SPAs.
        '--headless',
        '--window-size=1920,1080',
    ]);

    return RemoteWebDriver::create(
        'http://localhost:9515', DesiredCapabilities::chrome()->setCapability(
            ChromeOptions::CAPABILITY, $options
        )
    );
}
```