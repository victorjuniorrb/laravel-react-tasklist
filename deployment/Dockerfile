# Accepted values: 8.3 - 8.2
ARG PHP_VERSION=8.2
ARG FRANKENPHP_VERSION=1.3
ARG COMPOSER_VERSION=2.2
ARG NODE_VERSION=20-alpine

###########################################

FROM composer:${COMPOSER_VERSION} AS vendor

FROM dunglas/frankenphp:${FRANKENPHP_VERSION}-php${PHP_VERSION}-alpine as base

ARG WWWUSER=1000
ARG WWWGROUP=1000
ARG TZ=America/Sao_Paulo
ARG APP_DIR=/var/www/html


ENV TERM=xterm-color \
    WITH_HORIZON=false \
    WITH_SCHEDULER=false \
    OCTANE_SERVER=frankenphp \
    USER=octane \
    ROOT=${APP_DIR} \
    COMPOSER_FUND=0 \
    COMPOSER_MAX_PARALLEL_HTTP=24 \
    XDG_CONFIG_HOME=${APP_DIR}/.config \
    XDG_DATA_HOME=${APP_DIR}/.data

WORKDIR ${ROOT}

SHELL ["/bin/sh", "-eou", "pipefail", "-c"]

RUN ln -snf /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo ${TZ} > /etc/timezone

    RUN apk update; \
    apk upgrade; \
    apk add --no-cache \
    curl \
    wget \
    nano \
    ncdu \
    procps \
    ca-certificates \
    supervisor \
    libsodium-dev \
    # Install PHP extensions (included with dunglas/frankenphp)
    && install-php-extensions \
    bz2 \
    pcntl \
    mbstring \
    bcmath \
    sockets \
    pgsql \
    pdo_pgsql \
    opcache \
    exif \
    pdo_mysql \
    zip \
    intl \
    gd \
    redis \
    rdkafka \
    memcached \
    igbinary \
    ldap \
    swoole \
    && docker-php-source delete \
    && rm -rf /var/cache/apk/* /tmp/* /var/tmp/*

RUN arch="$(apk --print-arch)" \
    && case "$arch" in \
    armhf) _cronic_fname='supercronic-linux-arm' ;; \
    aarch64) _cronic_fname='supercronic-linux-arm64' ;; \
    x86_64) _cronic_fname='supercronic-linux-amd64' ;; \
    x86) _cronic_fname='supercronic-linux-386' ;; \
    *) echo >&2 "error: unsupported architecture: $arch"; exit 1 ;; \
    esac \
    && wget -q "https://github.com/aptible/supercronic/releases/download/v0.2.33/${_cronic_fname}" \
    -O /usr/bin/supercronic \
    && chmod +x /usr/bin/supercronic \
    && mkdir -p /etc/supercronic \
    && echo "*/1 * * * * php ${ROOT}/artisan schedule:run --no-interaction" > /etc/supercronic/laravel

RUN addgroup -g ${WWWGROUP} ${USER} \
    && adduser -D -h ${ROOT} -G ${USER} -u ${WWWUSER} -s /bin/sh ${USER}

RUN mkdir -p /var/log/supervisor /var/run/supervisor \
    && chown -R ${USER}:${USER} ${ROOT} /var/log /var/run \
    && chmod -R a+rw ${ROOT} /var/log /var/run

RUN cp ${PHP_INI_DIR}/php.ini-production ${PHP_INI_DIR}/php.ini

USER ${USER}

COPY --chown=${USER}:${USER} --from=vendor /usr/bin/composer /usr/bin/composer
COPY --chown=${USER}:${USER} composer.json composer.lock ./

RUN composer install \
    --no-dev \
    --no-interaction \
    --no-autoloader \
    --no-ansi

###########################################
# Build frontend assets with NPM
###########################################

FROM node:${NODE_VERSION} AS build

ENV ROOT=/var/www/html

ARG VITE_REVERB_APP_KEY
ARG VITE_REVERB_HOST
ARG VITE_REVERB_PORT
ARG VITE_REVERB_SCHEME
ARG VITE_REVERB_APP_DEV_KEY
ARG VITE_REVERB_DEV_HOST
ARG VITE_REVERB_DEV_PORT
ARG VITE_REVERB_DEV_SCHEME

WORKDIR ${ROOT}

RUN npm config set update-notifier false && npm set progress=false

COPY package*.json ./

RUN if [ -f $ROOT/package-lock.json ]; \
    then \
    npm ci --loglevel=error --no-audit; \
    else \
    npm install --loglevel=error --no-audit; \
    fi

COPY --from=base ${ROOT}/vendor vendor
COPY . .

RUN npm run build

###########################################

FROM base

ENV BROADCAST_CONNECTION=null
ENV BROADCAST_DRIVER=null

COPY --chown=${USER}:${USER} . .
COPY --chown=${USER}:${USER} --from=build ${ROOT}/public public

# LABEL org.opencontainers.image.source=https://github.com/cettufg/Sistema-Custos
# LABEL org.opencontainers.image.description="Imagem do Sistema Custos Produto"
# LABEL org.opencontainers.image.licenses=MIT

RUN mkdir -p \
    storage/framework/sessions \
    storage/framework/views \
    storage/framework/cache \
    storage/framework/testing \
    storage/logs \
    bootstrap/cache && chmod -R a+rw storage

COPY --chown=${USER}:${USER} deployment/supervisord.conf /etc/supervisor/
COPY --chown=${USER}:${USER} deployment/supervisord.frankenphp.conf /etc/supervisor/conf.d/
COPY --chown=${USER}:${USER} deployment/supervisord.*.conf /etc/supervisor/conf.d/
COPY --chown=${USER}:${USER} deployment/start-container /usr/local/bin/start-container
COPY --chown=${USER}:${USER} deployment/php.ini ${PHP_INI_DIR}/conf.d/99-octane.ini

# FrankenPHP embedded PHP configuration
COPY --chown=${USER}:${USER} deployment/php.ini /lib/php.ini

# Copy file with .env.exempla to .env on container layer to reduce ENV insercion on deploy
COPY --chown=${USER}:${USER} .env.example .env

RUN composer install \
    --classmap-authoritative \
    --no-interaction \
    --no-ansi \
    --no-dev \
    && composer clear-cache

RUN chmod +x /usr/local/bin/start-container

RUN cat deployment/utilities.sh >> ~/.bashrc

EXPOSE 8000
EXPOSE 8080
EXPOSE 443
EXPOSE 443/udp
EXPOSE 2019

ENTRYPOINT ["start-container"]

HEALTHCHECK --start-period=5s --interval=2s --timeout=5s --retries=8 CMD php artisan octane:status || exit 1
