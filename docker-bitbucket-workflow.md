```
# This is a sample build configuration for PHP.
# Check our guides at https://confluence.atlassian.com/x/e8YWN for more examples.
# Only use spaces to indent your .yml configuration.
# You can specify a custom docker image from Docker Hub as your build environment.
# run composer check-platform-reqs for a list of required extensions.
image: php:7.2-fpm
pipelines:
  default:
# Not needed unless you're doing feature tests.
#    - step:
#        name: Build
#        image: node:8.9.4
#        caches:
#          - node
#        script:
#          - npm install
#          - npm run prod
#        artifacts:
#          - public/**
    - step:
        name: Test
        caches:
          - composer
        artifacts:
          - storage/**
          - vendor/**
          - public/**
          - .env
        script:
          #Update Image
          - apt-get update

          #Install Zip
          - apt-get install -qy zlib1g-dev zip unzip
          - docker-php-ext-install zip

          #Install Git
          - apt-get install -qy git
          - apt-get install -qy curl

          #Install MySql
          - apt-get install -qy mysql-client
          - docker-php-ext-install pdo_mysql

          #Install Crypt
          - apt-get install -qy libmcrypt-dev
          - yes | pecl install mcrypt-1.0.1

          #Install Composer Platform Reqs
          - docker-php-ext-install bcmath

          #Copy Environment File
          - ln -f -s .env.pipelines .env

          #Install Composer
          - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
          - composer install

          #Migrate Database
          - php artisan key:generate
          - php artisan migrate

          #Run Test Suite
          - ./vendor/bin/phpunit --verbose
        services:
          #Add MySql Database
          - mysql
# Additional Manual Step for Deployer if Needed.
# Otherwise move "artisan deploy..." to testing step so deployment is automatic.
    - step:
        name: Deploy to Production
        deployment: production
        trigger: manual
        script:
          #Update Image
          - apt-get update

          #Install Zip
          - apt-get install -qy zlib1g-dev zip unzip
          - docker-php-ext-install zip

          #Install Git
          - apt-get install -qy git
          - apt-get install -qy curl

          #Install MySql
          - apt-get install -qy mysql-client
          - docker-php-ext-install pdo_mysql

          #Install Crypt
          - apt-get install -qy libmcrypt-dev
          - yes | pecl install mcrypt-1.0.1

          #Install Composer Platform Reqs
          - docker-php-ext-install bcmath

          #Copy Environment File
          - ln -f -s .env.pipelines .env

          #Install Composer
          - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
          - composer install

          #Migrate Database
          - php artisan key:generate
          #Deploy to Development
          #https://github.com/lorisleiva/laravel-deployer
          - php artisan deploy production -v
definitions:
  services:
    mysql:
      image: mysql:5.7
      environment:
        MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
        MYSQL_DATABASE: 'bitbucket'
        MYSQL_PASSWORD: 'bitbucket'
        MYSQL_USER: 'bitbucket'
```