# Out-of-the-box, Nginx with environment variables support 

Nginx that uses environment variables to build a configuration file using `envsubst`. I was tired of copying
and pasting this image template everywhere.

In addition, I could not agree with solutions like:

- https://hub.docker.com/r/martin/nginx/
- https://github.com/shiphp/nginx-env
- https://thepracticalsysadmin.com/templated-nginx-configuration-with-bash-and-docker/

Then decided to use what is recommended at [official nginx](https://hub.docker.com/_/nginx) with some help from 
[JavaCS3](https://github.com/JavaCS3) on [Github](https://github.com/docker-library/docs/issues/496)  

The **goal** is to never have to create build an image for Nginx only for changing the configuration file. However,
if you need to server static files you will have to deal with volumes, or you build your image to add files or 
add extra templates. 

There's a lot to do, but I will start lean and iterate on a per request basis.

# Supported tags and respective `Dockerfile` links

- [`latest`](https://github.com/milhomem/nginx/blob/master/alpine/base/Dockerfile)
- [`1.21-alpine`](https://github.com/milhomem/nginx/blob/master/alpine/base/Dockerfile)
- [`php`, `1.21-alpine-php`](https://github.com/milhomem/nginx/blob/master/alpine/php/Dockerfile)
- [`1.17-alpine-php-1.0`](https://github.com/milhomem/nginx/blob/e66e777ad9d4b648b480d7fc11a2903dd893f0d6/alpine/php/Dockerfile)

# What is nginx?

> [wikipedia.org/wiki/Nginx](https://en.wikipedia.org/wiki/Nginx)

![logo](https://raw.githubusercontent.com/docker-library/docs/01c12653951b2fe592c1f93a13b4e289ada0e3a1/nginx/logo.png)

# How to use this image

## Where it comes from? 

**FROM [nginx](https://hub.docker.com/_/nginx)**

## Changing the configuration

To change the configuration we use **Environment Variable**

```console
docker run --rm --name some-nginx -p 8080:80 -e NGINX_HOST=192.168.99.100.xip.io milhomem/nginx
```

Then you can hit `http://192.168.99.100.xip.io:8080` in your browser.

Another example:

```console
docker run --rm --name some-nginx -p 8080:80 -v /local/src:/var/www -e STATIC_FILES_ROOT=/var/www/public milhomem/nginx
```

And if you need to understand how the applied configuration looks like you can, in another console, run:

```console
docker exec some-nginx cat /etc/nginx/conf.d/default.conf
```

### Available environment variables 

Currently, these are the available variables, but they will depend on the tag you use:

- CLIENT_MAX_BODY_SIZE
- NGINX_PORT
- NGINX_HOST
- STATIC_FILES_ROOT
- FASTCGI_FILES_ROOT
- FASTCGI_HOST
- FASTCGI_PORT
- FASTCGI_CONNECT_TIMEOUT
- FASTCGI_SEND_TIMEOUT
- FASTCGI_READ_TIMEOUT
- PHP_VALUE

> If you have extra needs you can either file an issue or see how to create
a complex configuration below

## Complex configuration

Every template file in the config folder `/etc/nginx/conf.d/*.template` will be generated replacing all environment
variables and included in the Nginx configuration.

You can then create a complex configuration:

- `.conf.template` will be added under the `http` directive, allowing you to add multiple `servers` if needed.
- `.rules.template` will be added under the main `server` template.
- `.fastcgi_params.template` will be included together with the `fastcgi` proxy configurations. 

As an example, I can create my own Dockerfile to include extra rules and add my static files to be bundled together:

```dockerfile
FROM milhomem/nginx

ADD config/templates/*.rules.template /etc/nginx/conf.d/

ADD src .
```

> Note: Your templates can still use variables to be replaced by envvars, making your image as flexible as this one.

### Using environment variables in Nginx configuration

Out-of-the-box, Nginx doesn't support environment variables inside most configuration blocks, 
but `envsubst` may be used as a workaround if you need to generate your Nginx configuration dynamically 
before Nginx starts.

Here is a few examples using docker-compose.yml:

```yaml
version: '3'

services:
  nginx:
    image: milhomem/nginx:php
    ports:
     - "8080:80"
    environment:
       FASTCGI_HOST: "hello_world_app"
       FASTCGI_FILES_ROOT: "/app"
       
  hello_world_app:
    image: php:7.3-fpm
    command: |
          sh -c "
          mkdir /app;
          echo '<?php phpinfo(); ?>' > /app/index.php;
          php-fpm
          "
```

```yaml
version: '3'

services:
  web:
    image: milhomem/nginx:php
    ports:
     - "8080:8081"
    environment:
       NGINX_HOST: "localhost"
       NGINX_PORT: "8081"
       FASTCGI_FILES_ROOT: "/var/www"
       FASTCGI_HOST: "fcgi"
       PHP_VALUE: "date.timezone=GMT\nmax_execution_time=30"
       CLIENT_MAX_BODY_SIZE: "10m"
       FASTCGI_CONNECT_TIMEOUT: "5"
       FASTCGI_SEND_TIMEOUT: "5"
       FASTCGI_READ_TIMEOUT: "5"

  fcgi:
    image: php:7.3-fpm
    volumes:
      - "source_code:/var/www"
volumes:
  source_code:
    driver: local
```

# Image Variants

The `nginx` images come in many flavors, each designed for a specific use case.

## `milhomem/nginx:<version>-alpine-php`

This image is based on the popular [Alpine Linux project](http://alpinelinux.org), 
available in [the `alpine` official image](https://hub.docker.com/_/alpine). 
Alpine Linux is much smaller than most distribution base images (~5MB), and thus leads to much slimmer images 
in general.

It is the first use case for this Nginx image with a PHP FastCGI backend. It will proxy the requests 
to the backend that is linked via `FASTCGI_HOST`

## `milhomem/nginx:latest`

This image is based on the popular [Alpine Linux project](http://alpinelinux.org).

This image is just a server for static content, for cases where you just want to host your assets files.

In this scenario you must copy files to the image (build your own) or mount a volume in the container.

# Contribute

If you need more flavors that you know are very common, please file a new issue or create an PR.

Get in touch [@milmeninos](https://twitter.com/milmeninos) if you want to be part of the project, 
if you have suggestions please file an issue or create an PR.

# Building the images

We need to be mindful that people run docker in to multiple platforms and we will support all that our base
image supports.

To build the image run the following command:

```shell
docker buildx build \
  --push \
  --platform linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x,linux/386 \
  --tag milhomem/nginx:latest \
  --tag milhomem/nginx:1.21-alpine \
  alpine/base

docker buildx build \
  --push \
  --platform linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x,linux/386 \
  --tag milhomem/nginx:php \
  --tag milhomem/nginx:1.21-alpine-php \
  --tag milhomem/nginx:1.21-alpine-php-1.0 \
  alpine/php
```
