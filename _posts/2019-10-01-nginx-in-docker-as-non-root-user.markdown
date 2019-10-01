---
layout: post
title: Nginx in Docker as non root user
date: 2019-10-01 08:00:00 +0200
description: Ho to buid and run nginx as non root user
img: docker+nginx.png
tags: [docker, nginx, non-root]
---

By default docker containers run as root. I don`t like that.

## What you need to know

These are the things you need to keep in mind if you want to run nginx as non root user:

- non-root-user needs read access to web app files
- non-root-user needs read/write access to `/var/run/nginx.pid`
- non-root-user needs read/write access to `/var/cache/nginx`
- only root can listen on ports below 1024, so you will need to use higher-numbered ports (not 80 and 443). This is not an issue since you can map your host port to your container port

## Prepare user on docker host system

First, you need to create user and group on docker host:
{% highlight Bash %}
#create Group_Name with gid Group_ID
sudo groupadd -g Group_ID Group_Name

#create User_name with gid User_ID 
sudo adduser -u User_ID User_name 

#add User_name to Group_Name
sudo usermod -a -G Group_Name User_name
{% endhighlight %}
So in our case:
{% highlight Bash %}
sudo groupadd -g 1443 non-root-user-group
sudo adduser -u 1443 non-root-user
sudo usermod -a -G non-root-user-group non-root-user
{% endhighlight %}

## Prepare config files on docker host system

You need at least `nginx.conf` and `default.conf` to run nginx. You may want to start with the config files provided in the offical image. An easy way to copy the original files from the image to your host is to start a container and use docker cp:

{% highlight Bash %}
docker run --name nginx -d nginx:stable
docker cp nginx:/etc/nginx/ /dockervolumes/nginx-unprivileged/
docker stop nginx
docker rm nginx
{% endhighlight %}

From file `nginx.conf` remove line
{% highlight Bash %}
user nginx;
{% endhighlight %}

And change ports in `default.conf` (and in all vhost.conf files) from 80 to 8080 and from 443 to 8443. For exmaple `default.conf` may look like this:

{% highlight Nginx %}
server {
    listen 8080;
    server_name localhost;
    location / {
        root /var/www;
        index index.html index.htm;
    }
}
{% endhighlight %}

## Dockerfile

Here is an example of `Dockerfile` from official nginx image with modified user and rights.

{% highlight Bash %}
ARG IMAGE=alpine:3.10
FROM \$IMAGE

LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"

ENV NGINX_VERSION 1.17.3
ENV NJS_VERSION 0.3.5
ENV PKG_RELEASE 1

RUN set -x \

# create nginx user/group first, to be consistent throughout docker variants

    && addgroup -g 1433 -S nginx \
    && adduser -S -D -H -u 1443 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx \
    && apkArch="$(cat /etc/apk/arch)" \
    && nginxPackages=" \
        nginx=${NGINX_VERSION}-r${PKG_RELEASE} \
        nginx-module-xslt=${NGINX_VERSION}-r${PKG_RELEASE} \
        nginx-module-geoip=${NGINX_VERSION}-r${PKG_RELEASE} \
        nginx-module-image-filter=${NGINX_VERSION}-r${PKG_RELEASE} \
        nginx-module-njs=${NGINX_VERSION}.${NJS_VERSION}-r${PKG_RELEASE} \
    " \
    && case "$apkArch" in \
        x86_64) \

# arches officially built by upstream

            set -x \
            && KEY_SHA512="e7fa8303923d9b95db37a77ad46c68fd4755ff935d0a534d26eba83de193c76166c68bfe7f65471bf8881004ef4aa6df3e34689c305662750c0172fca5d8552a *stdin" \
            && apk add --no-cache --virtual .cert-deps \
                openssl \
            && wget -O /tmp/nginx_signing.rsa.pub https://nginx.org/keys/nginx_signing.rsa.pub \
            && if [ "$(openssl rsa -pubin -in /tmp/nginx_signing.rsa.pub -text -noout | openssl sha512 -r)" = "$KEY_SHA512" ]; then \
                echo "key verification succeeded!"; \
                mv /tmp/nginx_signing.rsa.pub /etc/apk/keys/; \
            else \
                echo "key verification failed!"; \
                exit 1; \
            fi \
            && printf "%s%s%s\n" \
                "https://nginx.org/packages/mainline/alpine/v" \
                `egrep -o '^[0-9]+\.[0-9]+' /etc/alpine-release` \
                "/main" \
            | tee -a /etc/apk/repositories \
            && apk del .cert-deps \
            ;; \
        *) \

# we're on an architecture upstream doesn't officially build for

# let's build binaries from the published packaging sources

            set -x \
            && tempDir="$(mktemp -d)" \
            && chown nobody:nobody $tempDir \
            && apk add --no-cache --virtual .build-deps \
                gcc \
                libc-dev \
                make \
                openssl-dev \
                pcre-dev \
                zlib-dev \
                linux-headers \
                libxslt-dev \
                gd-dev \
                geoip-dev \
                perl-dev \
                libedit-dev \
                mercurial \
                bash \
                alpine-sdk \
                findutils \
            && su nobody -s /bin/sh -c " \
                export HOME=${tempDir} \
                && cd ${tempDir} \
                && hg clone https://hg.nginx.org/pkg-oss \
                && cd pkg-oss \
                && hg up ${NGINX_VERSION}-${PKG_RELEASE} \
                && cd alpine \
                && make all \
                && apk index -o ${tempDir}/packages/alpine/${apkArch}/APKINDEX.tar.gz ${tempDir}/packages/alpine/${apkArch}/*.apk \
                && abuild-sign -k ${tempDir}/.abuild/abuild-key.rsa ${tempDir}/packages/alpine/${apkArch}/APKINDEX.tar.gz \
                " \
            && echo "${tempDir}/packages/alpine/" >> /etc/apk/repositories \
            && cp ${tempDir}/.abuild/abuild-key.rsa.pub /etc/apk/keys/ \
            && apk del .build-deps \
            ;; \
    esac \
    && apk add --no-cache $nginxPackages \

# if we have leftovers from building, let's purge them (including extra, unnecessary build deps)

    && if [ -n "$tempDir" ]; then rm -rf "$tempDir"; fi \
    && if [ -n "/etc/apk/keys/abuild-key.rsa.pub" ]; then rm -f /etc/apk/keys/abuild-key.rsa.pub; fi \
    && if [ -n "/etc/apk/keys/nginx_signing.rsa.pub" ]; then rm -f /etc/apk/keys/nginx_signing.rsa.pub; fi \

# remove the last line with the packages repos in the repositories file

    && sed -i '$ d' /etc/apk/repositories \

# Bring in gettext so we can get `envsubst`, then throw

# the rest away. To do this, we need to install `gettext`

# then move `envsubst` out of the way so `gettext` can

# be deleted completely, then move `envsubst` back.

    && apk add --no-cache --virtual .gettext gettext \
    && mv /usr/bin/envsubst /tmp/ \
    \
    && runDeps="$( \
        scanelf --needed --nobanner /tmp/envsubst \
            | awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
            | sort -u \
            | xargs -r apk info --installed \
            | sort -u \
    )" \
    && apk add --no-cache $runDeps \
    && apk del .gettext \
    && mv /tmp/envsubst /usr/local/bin/ \

# Bring in tzdata so users could set the timezones through the environment

# variables

    && apk add --no-cache tzdata

# implement changes required to run NGINX as an unprivileged user

RUN sed -i -e '/listen/!b' -e '/80;/!b' -e 's/80;/8080;/' /etc/nginx/conf.d/default.conf \
 && sed -i -e '/user/!b' -e '/nginx/!b' -e '/nginx/d' /etc/nginx/nginx.conf \
 && sed -i 's!/var/run/nginx.pid!/tmp/nginx.pid!g' /etc/nginx/nginx.conf \
 && sed -i "/^http {/a \ proxy_temp_path /tmp/proxy_temp;\n client_body_temp_path /tmp/client_temp;\n fastcgi_temp_path /tmp/fastcgi_temp;\n uwsgi_temp_path /tmp/uwsgi_temp;\n scgi_temp_path /tmp/scgi_temp;\n" /etc/nginx/nginx.conf

RUN chown -R 1443:0 /var/cache/nginx \
 && chmod -R g+w /var/cache/nginx \
 && touch /var/run/nginx.pid \
 && chown -R 1443:0 /var/run/nginx.pid \
 && ln -sf /dev/stdout /var/log/nginx/access.log \
 && ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 8080 8443

STOPSIGNAL SIGTERM

USER 1443

CMD ["nginx", "-g", "daemon off;"]
{% endhighlight %}

## docker-compose.yaml

{% highlight YAML %}
version: "3.2"

services:
  nginx-unprivileged:
    build:
      context: ./nginx-unprivileged
      dockerfile: Dockerfile
    image: yoyanec/nginx-unprivileged:1.17.3
    container_name: nginx-unprivileged
    volumes: 
      - /dockervolumes/nginx-unprivileged/nginx:/etc/nginx:ro 
      - /dockervolumes/nginx-unprivileged/www:/var/www:ro
    ports: 
      - "80:8080" 
      - "443:8443"
    networks: 
      - nginx-unprivileged-network
    restart: unless-stopped
    environment: 
      - TZ=Europe/Bratislava
{% endhighlight %}

## How to start unprivileged nginx service
All you need to do is:

{% highlight Bash %}
sudo docker-compose up -d nginx-unprivileged
{% endhighlight %}