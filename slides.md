# Dockerize Legacy Software

* Choose base image
* Dockerfile code style
* Install dependencies on build time
* Uses process supervision tool
* Configuration
* Logging
* Writing files
* Mount directory not files
* File permission in docker run (not docker build)

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
RUN php /usr/local/bin/composer global require 'hirak/prestissimo'
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
        && php /usr/local/bin/composer global require 'hirak/prestissimo'
```

## Install dependencies on build time

1. Speed up running time.
2. Docker image that can be running anywhere

Ruby example:
```
WORKDIR /var/www/html/legacyapp
COPY Gemfile .
RUN bundle install
```

## Uses process supervision tool

- Run multiple applications in a single container
- Forward signals
  * Most software dosn't handle signals well when running as PID 1
- Application availability
- Execute scripts before running any application

Supervision tools that we used:
- s6-overylay https://github.com/just-containers/s6-overlay
- dumb-init https://github.com/Yelp/dumb-init

s6-overylay example:
```
ADD https://github.com/just-containers/s6-overlay/releases/download/v1.18.1.5/s6-overlay-amd64.tar.gz /tmp/
RUN tar xzf /tmp/s6-overlay-amd64.tar.gz -C /
CMD ["/init"]
...
...
COPY etc/services.d /etc/
```

dumb-init example:
```
RUN wget -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.0/dumb-init_1.2.0_amd64
RUN chmod +x /usr/local/bin/dumb-init
CMD ["/usr/local/bin/dumb-init", "node", "/var/www/html/server.js"]
```

## Configuration

For single file configuration, use `confd` tool to update using environment.

Dockerfile
```
ADD https://github.com/kelseyhightower/confd/releases/download/v0.11.0/confd-0.11.0-linux-amd64 /usr/bin/confd
RUN chmod +x /usr/bin/confd
...
ENV database_host=localhost
...
ADD ./etc/cont-init.d /etc/cont-init.d
ADD ./etc/confd /etc/confd
```

cont-init.d/01-generate-conf.sh
```
#!/usr/bin/with-contenv sh

confd -onetime -backend env
```

## Logging

- log to stdout stderr
- log to file
- log to syslog socket

Share syslog socket with container (docker-compose.yml):
```
  ...
  legacy-service:
    image: <docker_image>
    volumes:
      - /dev/log:/dev/log
  ...
```

stdout/stderr (Official Nginx Dockerfile):
```
        ...
        # forward request and error logs to docker log collector
        && ln -sf /dev/stdout /var/log/nginx/access.log \
        && ln -sf /dev/stderr /var/log/nginx/error.log
        ...
```

log to file `/var/log/legacyapp/api.log`:
```
  ...
  legacy-service:
    image: <docker_image>
    volumes:
      - /var/log/legacyapp:/var/log/legacyapp
  ...
```

## Writing files

If software write files to disk make sure you mount the directory
with docker host.

Dont write any files inside container:
- Container has default size limit (currently 10GB)
- re-creating container will delete the files

Share syslog socket with container (docker-compose.yml):
```
  ...
  legacy-service:
    image: <docker_image>
    volumes:
      - /var/www/html/legacyapp/storage:/opt/legacyapp/storage
  ...
```

We're not using any data container.

## Mount directory not files

- If mount file not exists docker will create an empty directory
- Older docker version doesn't update mounted files

## File permission

Don't change file permission on Dockerfile, it will duplicate file size.

```
$ dd if=/dev/zero of=./testfile bs=1M count=200
200+0 records in
200+0 records out
209715200 bytes (210 MB, 200 MiB) copied, 0.127021 s, 1.7 GB/s
```

Dockerfile
```
FROM alpine

ADD testfile /
RUN chown 2:2 testfile
ADD testfile /abc
```

docker build
```
$ docker build -t tmp .
Sending build context to Docker daemon  209.7MB
Step 1/4 : FROM alpine
 ---> 88e169ea8f46
Step 2/4 : ADD testfile /
 ---> ecbd403a6d91
Removing intermediate container b7f1b14d3c63
Step 3/4 : RUN chown 2:2 testfile
 ---> Running in fdcaeb3ca7e8
 ---> 115bf649e87a
Removing intermediate container fdcaeb3ca7e8
Step 4/4 : ADD testfile /abc
 ---> 4f0e9dc7d459
Removing intermediate container d17bcb1dde1b
Successfully built 4f0e9dc7d459
Successfully tagged tmp:latest
```

docker history
```
docker history -H tmp
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
4f0e9dc7d459        14 seconds ago      /bin/sh -c #(nop) ADD file:c17c4642cd3cefb...   210MB
115bf649e87a        20 seconds ago      /bin/sh -c chown 2:2 testfile                   210MB
ecbd403a6d91        28 seconds ago      /bin/sh -c #(nop) ADD file:c17c4642cd3cefb...   210MB
88e169ea8f46        4 months ago        /bin/sh -c #(nop) ADD file:92ab746eb22dd3e...   3.98MB
```

### The Solution

if you're using `s6-overlay`, add a script to change permission

etc/cont-init.d/03-fix-webroot-ownership.sh
```
#!/bin/sh
set -ex

chown -R apache:apache /var/www/html/
```
