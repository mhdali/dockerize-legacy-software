# Dockerize Legacy Software

* Choose base image
* Dockerfile code style
* Install dependencies while building
* Uses process supervision tool
* configuration
* logging
* writing files (container size limit, share directory with host)
* mount directory not files
* Install packages from private repositories
* file permission in docker run (not dokcer build)
* Directory structure share with docker host
  - use base images matches your production OS then optimize

## Start matches production environment

If your software is running on a specific operating system,
Use docker base image matches the production environment.

Ubuntu example:
A platform running on `Ubuntu 14.04`, your Dockerfile will be:
```
FROM ubuntu:14.04
...
```

## Dockerfile code style

```
RUN apk add --no-cache php5 php5-phar php5-json php5-openssl php5-pdo php5-pdo_mysql php5-curl php5-ctype php5-dom php5-apache2 php5-mysql php5-mysqli
RUN mkdir -p /var/www/html/legacyapp
RUN chown www-data:www-data /var/www/html/legacyapp
```
VS
```
RUN apk add --no-cache \
                php5 \
                php5-phar \
                php5-json \
                php5-openssl \
                php5-pdo \
                php5-pdo_mysql \
                php5-curl \
                php5-ctype \
                php5-dom \
                php5-apache2 \
                php5-mysql \
                php5-mysqli
        && mkdir -p /var/www/html/legacyapp \
        && chown www-data:www-data /var/www/html/legacyapp
```

## Install dependencies while building

1. Speed up running time.
2. Docker image that can be running anywhere

Ruby example:
```
WORKDIR /var/www/html/legacyapp
COPY Gemfile .
RUN bundle install
```

## Uses process supervision tool

- Run multiple application in a single container
- Make sure forward kernel signals
  * Most software dosn't handle signals while running as PID 1
- Application availability
- execute scripts before running any application

s6-overylay example:
```
ADD https://github.com/just-containers/s6-overlay/releases/download/v1.18.1.5/s6-overlay-amd64.tar.gz /tmp/
RUN tar xzf /tmp/s6-overlay-amd64.tar.gz -C /
CMD ["/init"]
...
...
COPY etc/services.d /etc/
```

## Configuration

```
ADD https://github.com/kelseyhightower/confd/releases/download/v0.11.0/confd-0.11.0-linux-amd64 /usr/bin/confd
RUN chmod +x /usr/bin/confd
```
