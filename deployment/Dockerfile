# Create args for PHP extensions and PECL packages we need to install.
# This makes it easier if we want to install packages,
# as we have to install them in multiple places.
# This helps keep ou Dockerfiles DRY -> https://bit.ly/dry-code
# You can see a list of required extensions for Laravel here: https://laravel.com/docs/8.x/deployment#server-requirements
ARG PHP_EXTS="bcmath ctype fileinfo mbstring pdo pdo_mysql dom pcntl"
ARG PHP_PECL_EXTS="redis"
ARG APP_ROOT="/var/www/html"

# use --no-cache for disable cache on php build
ARG NO_CACHE=""

# Env to use on label and container description
ARG LABEL_URL="https://github.com/victorjuniorrb/laravel-react-tasklist"
ARG LABEL_DESC="Laravel React Tasklist"

#------------------------------------------------------------------------------------------
# We need to build the Composer base to reuse packages we've installed
FROM composer:2.7 AS composer_base

# We need to declare that we want to use the args in this build step
ARG PHP_EXTS
ARG PHP_PECL_EXTS
ARG NO_CACHE
ARG APP_ROOT

# First, create the application directory, and some auxilary directories for scripts and such
RUN mkdir -p ${APP_ROOT} \
    ${APP_ROOT}/bin \
    ${APP_ROOT}/bootstrap/cache \
    ${APP_ROOT}/storage/framework/sessions \
    ${APP_ROOT}/storage/framework/views \
    ${APP_ROOT}/storage/framework/cache \
    ${APP_ROOT}/storage/framework/testing \
    ${APP_ROOT}/storage/logs

# Next, set our working directory
WORKDIR ${APP_ROOT}

# We need to create a composer group and user, and create a home directory for it, so we keep the rest of our image safe,
# And not accidentally run malicious scripts
RUN addgroup -S composer \
    && adduser -S composer -G composer \
    && chown -R composer ${APP_ROOT} \
    && apk add --virtual build-dependencies ${NO_CACHE} ${PHPIZE_DEPS} openssl ca-certificates libxml2-dev oniguruma-dev \
    && docker-php-ext-install -j$(nproc) ${PHP_EXTS} \
    && pecl install ${PHP_PECL_EXTS} \
    && docker-php-ext-enable ${PHP_PECL_EXTS} \
    && apk del build-dependencies

# Next we want to switch over to the composer user before running installs.
# This is very important, so any extra scripts that composer wants to run,
# don't have access to the root filesystem.
# This especially important when installing packages from unverified sources.
USER composer

# Copy in our dependency files.
# We want to leave the rest of the code base out for now,
# so Docker can build a cache of this layer,
# and only rebuild when the dependencies of our application changes.
COPY --chown=composer composer.json composer.lock ./

# Install all the dependencies without running any installation scripts.
# We skip scripts as the code base hasn't been copied in yet and script will likely fail,
# as `php artisan` available yet.
# This also helps us to cache previous runs and layers.
# As long as comoser.json and composer.lock doesn't change the install will be cached.
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

# Copy file with .env.exempla to .env on container layer to reduce ENV insercion on deploy
COPY --chown=composer .env.example .env
# Copy in our actual source code so we can run the installation scripts we need
# At this point all the PHP packages have been installed,
# and all that is left to do, is to run any installation scripts which depends on the code base
COPY --chown=composer . .

# Now that the code base and packages are all available,
# we can run the install again, and let it run any install scripts.
RUN composer install --no-dev --prefer-dist

#------------------------------------------------------------------------------------------
# For the frontend, we want to get all the Laravel files,
# and run a production compile
FROM node:20 AS frontend

# We need to declare that we want to use the args in this build step
ARG APP_ROOT

# We need to copy in the Laravel files to make everything is available to our frontend compilation
COPY --from=composer_base ${APP_ROOT} ${APP_ROOT}

WORKDIR ${APP_ROOT}

# We want to install all the NPM packages,
# and compile the MIX bundle for production
RUN npm install && \
    npm run build

#------------------------------------------------------------------------------------------
# For running things like migrations, and queue jobs,
# we need a CLI container.
# It contains all the Composer packages,
# and just the basic CLI "stuff" in order for us to run commands,
# be that queues, migrations, tinker etc.
FROM php:8.2-alpine AS cli

LABEL org.opencontainers.image.source=${LABEL_URL}
LABEL org.opencontainers.image.description=${LABEL_DESC}
LABEL org.opencontainers.image.licenses=MIT
# LABEL org.opencontainers.image.vendor=CETT/UFG

# We need to declare that we want to use the args in this build step
ARG PHP_EXTS
ARG PHP_PECL_EXTS
ARG NO_CACHE
ARG APP_ROOT

WORKDIR ${APP_ROOT}

# We need to install some requirements into our image,
# used to compile our PHP extensions, as well as install all the extensions themselves.
# You can see a list of required extensions for Laravel here: https://laravel.com/docs/8.x/deployment#server-requirements
RUN apk add --virtual build-dependencies ${NO_CACHE} ${PHPIZE_DEPS} tzdata openssl ca-certificates libxml2-dev oniguruma-dev && \
    docker-php-ext-install -j$(nproc) ${PHP_EXTS} && \
    pecl install ${PHP_PECL_EXTS} && \
    docker-php-ext-enable ${PHP_PECL_EXTS} && \
    apk del build-dependencies
ENV TZ=America/Sao_Paulo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Next we have to copy in our code base from our initial build which we installed in the previous stage
COPY --from=composer_base ${APP_ROOT} ${APP_ROOT}
COPY --from=frontend ${APP_ROOT}/public ${APP_ROOT}/public

#------------------------------------------------------------------------------------------
# We need a stage which contains FPM to actually run and process requests to our PHP application.
FROM php:8.2-fpm-alpine AS fpm_server

LABEL org.opencontainers.image.source=${LABEL_URL}
LABEL org.opencontainers.image.description=${LABEL_DESC}
LABEL org.opencontainers.image.licenses=MIT
# LABEL org.opencontainers.image.vendor=CETT/UFG

# We need to declare that we want to use the args in this build step
ARG PHP_EXTS
ARG PHP_PECL_EXTS
ARG NO_CACHE
ARG APP_ROOT

WORKDIR ${APP_ROOT}


RUN apk add --virtual build-dependencies ${NO_CACHE} ${PHPIZE_DEPS} tzdata openssl ca-certificates libxml2-dev oniguruma-dev && \
    docker-php-ext-install -j$(nproc) ${PHP_EXTS} && \
    pecl install ${PHP_PECL_EXTS} && \
    docker-php-ext-enable ${PHP_PECL_EXTS} && \
    apk del build-dependencies
ENV TZ=America/Sao_Paulo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# As FPM uses the www-data user when running our application,
# we need to make sure that we also use that user when starting up,
# so our user "owns" the application when running
USER  www-data

# Copy custom configurations PHP
COPY deployment/php/custom.ini /usr/local/etc/php/conf.d/custom.ini

# We have to copy in our code base from our initial build which we installed in the previous stage
COPY --from=composer_base --chown=www-data ${APP_ROOT} ${APP_ROOT}
COPY --from=frontend --chown=www-data ${APP_ROOT}/public ${APP_ROOT}/public

# We want to cache the event, routes, and views so we don't try to write them when we are in Kubernetes.
# Docker builds should be as immutable as possible, and this removes a lot of the writing of the live application.
RUN php artisan event:cache && \
    # php artisan route:cache && \
    php artisan view:cache

#------------------------------------------------------------------------------------------
# We need an nginx container which can pass requests to our FPM container,
# as well as serve any static content.
FROM nginx:1.27-alpine AS web_server

LABEL org.opencontainers.image.source=${LABEL_URL}
LABEL org.opencontainers.image.description=${LABEL_DESC}
LABEL org.opencontainers.image.licenses=MIT
# LABEL org.opencontainers.image.vendor=CETT/UFG

# We need to declare that we want to use the args in this build step
ARG APP_ROOT

WORKDIR ${APP_ROOT}

# Trocar uid e gid do nginx para 82, igualando com o www-data para contornar falhar na criação 
# dos diretórios pelo laravel com permissão chmod 700.
RUN apk add --virtual build-dependencies ${NO_CACHE} shadow tzdata && \
    usermod -u 82 nginx && \
    apk del shadow
ENV TZ=America/Sao_Paulo
ENV APP_ROOT=${APP_ROOT}
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# We need to add our NGINX template to the container for startup,
# and configuration.
COPY deployment/nginx/default.conf.template /etc/nginx/templates/default.conf.template

# Copy in ONLY the public directory of our project.
# This is where all the static assets will live, which nginx will serve for us.
COPY --from=frontend --chown=nginx:nginx ${APP_ROOT}/public ${APP_ROOT}/public

# Enable storage/app/public folder for webserver
RUN mkdir -p ${APP_ROOT}/storage/app/public && \
    ln -snf ${APP_ROOT}/storage/app/public/ ${APP_ROOT}/public/storage

#------------------------------------------------------------------------------------------
# We need a CRON container to the Laravel Scheduler.
# We'll start with the CLI container as our base,
# as we only need to override the CMD which the container starts with to point at cron
FROM cli AS cron

LABEL org.opencontainers.image.source=${LABEL_URL}
LABEL org.opencontainers.image.description=${LABEL_DESC}
LABEL org.opencontainers.image.licenses=MIT
# LABEL org.opencontainers.image.vendor=CETT/UFG

# We need to declare that we want to use the args in this build step
ARG APP_ROOT

WORKDIR ${APP_ROOT}

# We want to create a laravel.cron file with Laravel cron settings, which we can import into crontab,
# and run crond as the primary command in the forground
RUN touch laravel.cron && \
    echo "* * * * * cd ${APP_ROOT} && php artisan schedule:run" >> laravel.cron && \
    crontab laravel.cron

CMD ["crond", "-l", "2", "-f"]

#------------------------------------------------------------------------------------------
# default stage
FROM cli
