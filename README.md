# Supported tags and respective `Dockerfile` links

- [`2.0.3-apache`, `2.0-apache`, `latest` (*2.0/apache/Dockerfile*)](https://github.com/codemix/yii2-dockerbase/blob/master/2.0/apache/Dockerfile)
- [`2.0.3-php-fpm`, `2.0-php-fpm`, `php-fpm` (*2.0/php-fpm/Dockerfile*)](https://github.com/codemix/yii2-dockerbase/blob/master/2.0/php-fpm/Dockerfile)
- [`2.0.3-hhvm`, `2.0-hhvm`, `hhvm` (*2.0/php-fpm/Dockerfile*)](https://github.com/codemix/yii2-dockerbase/blob/master/2.0/hhvm/Dockerfile)

Yii 2 Base
==========

This is a base image for Yii 2 projects.

> **IMPORTANT: The image does *not* contain an app template!**
> So you must always first combine it with your own application code to make it work!

The main purpose of this image is,

 * to provide a PHP runtime environment that is configured for Yii and
 * that has the base yii2 composer packages pre-installed.

## 1. Available versions

There are three flavours of this image

 * **Apache with PHP module** (based on `php:5.6.6-apache`)
 * **PHP-FPM** (based on `php:5.6.6-fpm`)
 * **HHVM** (based on `estebanmatias92/hhvm:3.5.1-fastcgi`)

## 2. How to use this Image

For all available flavours we recommend to start with our
[yii2-dockerized](https://github.com/codemix/yii2-dockerized) application template.
It comes with a ready to use `Dockerfile` and exemplifies how this base image is meant
to be used.

If you don't want to use that base template you can still build an application
from scratch. But still we recommend to study that template first.

Before you build your own application template, you should understand the basic
idea behind this image:

 * The image has all yii2 related composer packages pre-installed under `/var/www/vendor`.
 * The application code is expected under `/var/www/html`, with
   the public directory being `/var/www/html/web`.
 * You will *never* install any composer packages locally, but
   always into your container. If you do so, this will either override
   or add more packages to those already contained in this image.

You'll always need to have some application code available locally. To start, you
could use the official base image:

```
composer create-project --no-install yiisoft/yii2-app-basic
```

> **Note:** You need to fix the paths to `autoload.php` and `Yii.php` in the
> `index.php` file and also add a `'vendorPath' => '/var/www/vendor'` option
> in the `config/web.php`.


### 2.1 Using the Apache Variant

Create a `Dockerfile` in your application directory:

```
FROM codemix/yii2-base:latest

# Copy your app's source code into the container
COPY . /var/www/html
```

and a `docker-compose.yml`:

```
web:
    build: ./
    ports:
        - "8080:80"
    expose:
        - "80"
    volumes:
        - ./:/var/www/html/
```

Now you're ready to run `docker-compose up` to start your app. It should
be available from `http://localhost:8080` or your boot2docker VM if you use that.


### 2.2 Using the PHP-FPM Or HHVM Flavour

Create a `Dockerfile` in your application directory:

```
FROM codemix/yii2-base:php-fpm
# Or for HHVM:
#FROM codemix/yii2-base:hhvm

# Copy your app's source code into the container
COPY . /var/www/html
```

For this variant, you'll also need an nginx container. We have included
an example configuration with a `Dockerfile` in the image. You can copy
it from the container with:

```
docker create -name temp codemix/yii2-base:php-fpm
# Or for HHVM:
#docker create -name temp codemix/yii2-base:hhvm
docker cp temp:/opt/nginx/ .
docker rm temp
```

Finally create a `docker-compose.yml`:

```
app:
    build: ./
    expose:
        - "9000"
    volumes:
        - ./:/var/www/html/
nginx:
    build: ./nginx
    ports:
        - "8080:80"
    links:
        - app
    volumes_from:
        - app
```

Now you're ready to run `docker-compose up` to start your app. It should
be available from `http://localhost:8080` or your boot2docker VM if you use that.


## 3. How to customize the setup

### 3.1 Adding Composer Packages

To add composer packages, you need to provide a `composer.json` with
some modifications:

```
{
  "require": {
    "php": ">=5.4.0",
    "yiisoft/yii2": "2.0.3",
    "yiisoft/yii2-bootstrap": "2.0.3",
    "yiisoft/yii2-swiftmailer": "2.0.3"
  },
  "require-dev": {
    "yiisoft/yii2-debug": "2.0.3",
    "yiisoft/yii2-gii": "2.0.3"
  },
  "config": {
    "process-timeout": 1800,
    "vendor-dir": "/var/www/vendor"
  },
  "extra": {
    "asset-installer-paths": {
      "npm-asset-library": "../vendor/npm",
      "bower-asset-library": "../vendor/bower"
    }
  }
}
```

Note the `vendor-dir` configuration, which is crucial for this setup. It's also
important, that the versions there match those of this image. Otherwhise you again
loose the advantage of reusing the composer packages contained in the image.

You also have to map the local directory into the container in your `docker-compose.yml`.
If you have problems with githubs rate limit, you can provide a github API token.

```
web:
    build: ./
    ports:
        - "8080:80"
    expose:
        - "80"
    volumes:
        - ./:/var/www/html/
    environment:
        API_TOKEN: "<YOUR GITHUB API TOKEN>"
```

Now you can run the bundled `composer` command in your container.

```
docker-compose run --rm web compose update myrepo/mypackage
```

#### 3.2 Adding PHP Extensions

As the `apache` and `php-fpm` flavours extend from the official [php](https://registry.hub.docker.com/u/library/php/)
image, you can use `docker-php-ext-install` in your `Dockerfile`. You may also have to install
some required packages with `apt-get install` first. Here's an example:

```
RUN apt-get update \
    && apt-get -y install \
            libfreetype6-dev \
            libjpeg62-turbo-dev \
            libmcrypt-dev \
            libpng12-dev \
        --no-install-recommends \
    && rm -r /var/lib/apt/lists/* \
    && docker-php-ext-install iconv mcrypt \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install gd
```

#### 3.3 Adding HHVM Extensions

Please check the [hhvm](https://registry.hub.docker.com/u/estebanmatias92/hhvm/) base image for details.

