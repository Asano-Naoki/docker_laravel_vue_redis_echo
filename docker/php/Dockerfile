FROM php:8.0-fpm-alpine

ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

RUN apk add --no-cache git && \
    git clone https://github.com/phpredis/phpredis.git /usr/src/php/ext/redis && \
    chmod +x /usr/local/bin/install-php-extensions && \
    install-php-extensions pdo_mysql redis
