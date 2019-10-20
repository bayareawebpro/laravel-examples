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
```
name: ci
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      #- uses: actions/setup-node@v1
      #  with:
      #    node-version: 12.x
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1
      - name: Composer Install
        run: composer install --no-ansi --no-interaction --no-scripts --no-suggest --no-progress --prefer-dist
      - name: Lint
        run: php7.3 vendor/bin/phpstan analyse
      - name: Unit Test
        run: php7.3 vendor/bin/phpunit
      #- name: NPM Install
      #  run: npm install
      #- name: Mix Compile
      #  run: npm run production
      #- name: Dusk Driver
      #  run: php7.3 artisan dusk:chrome-driver
      #- name: Browser Test
      #  run: php7.3 artisan dusk
```


## Docker Workflow
- If you require custom extensions...
```
name: tests
on: [push, pull_request]
jobs:
  phpunit:
    runs-on: ubuntu-latest
    container:
      image: your/laravel-docker
    steps:
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1
      - name: Install Composer Dependencies
        run: |
          composer install --no-ansi --no-interaction --no-scripts --no-suggest --no-progress --prefer-dist
      - name: Lint
        run: |
          vendor/bin/phpstan analyse
      - name: Test
        run: |
          vendor/bin/phpunit
```