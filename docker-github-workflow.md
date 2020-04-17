Add to `.github/workflows/test.yaml`

Badges
```
![](https://github.com/my-account/my-repo/workflows/my-workflow-name/badge.svg)
![](https://img.shields.io/badge/License-MIT-success.svg)
![](https://img.shields.io/badge/Version-1.0-blue.svg)
```
## Package Workflow
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
        run: composer install --no-ansi --no-interaction --no-scripts --no-suggest --no-progress --prefer-dist
      - name: Lint
        run: php7.3 vendor/bin/phpstan analyse
      - name: Testsuite
        run: php7.3 vendor/bin/phpunit
```

## App Workflow

> Comment Out --disable-gpu if your app uses animations.
```
name: ci
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
          composer install --no-progress --no-suggest --prefer-dist --optimize-autoloader
          php artisan key:generate

      - name: Create Sqlite Database
        run: php artisan make:sqlite

      - name: Run PHP Analysis
        run: composer lint

      - name: Run PHP Tests
        run: composer test-unit

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


## Docker Workflow
- If you require custom extensions...
```
name: tests
on: [push, pull_request]
jobs:
  CI:
    runs-on: ubuntu-latest
    container:
      image: your/laravel-docker
    steps:
     ...
```