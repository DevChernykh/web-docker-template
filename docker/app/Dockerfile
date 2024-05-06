# ---------------------------------------------- Build Time Arguments --------------------------------------------------
ARG PHP_VERSION="8.3.6"
ARG PHP_ALPINE_VERSION="3.18"
ARG COMPOSER_VERSION="2.7.4"
ARG XDEBUG_VERSION="3.3.2"
ARG COMPOSER_AUTH
ARG APP_BASE_DIR="."

# -------------------------------------------------- Composer Image ----------------------------------------------------

FROM composer:${COMPOSER_VERSION} as composer

# ======================================================================================================================
#                                                   --- Base ---
# ---------------  This stage install needed extenstions, plugins and add all needed configurations  -------------------
# ======================================================================================================================

FROM php:${PHP_VERSION}-fpm-alpine${PHP_ALPINE_VERSION} AS base

# Required Args ( inherited from start of file, or passed at build )
ARG XDEBUG_VERSION

# Set SHELL flags for RUN commands to allow -e and pipefail
# Rationale: https://github.com/hadolint/hadolint/wiki/DL4006
SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

# ------------------------------------- Install Packages Needed Inside Base Image --------------------------------------
RUN echo -e "https://nl.alpinelinux.org/alpine/v3.18/main/\nhttps://nl.alpinelinux.org/alpine/v3.18/community/" > /etc/apk/repositories

RUN tail /etc/apk/repositories

RUN RUNTIME_DEPS="tini fcgi tzdata"; \
    SECURITY_UPGRADES="curl"; \
    apk add --no-cache --upgrade ${RUNTIME_DEPS} ${SECURITY_UPGRADES}

# ---------------------------------------- Install / Enable PHP Extensions ---------------------------------------------
RUN apk add --no-cache --virtual .build-deps \
      $PHPIZE_DEPS   \
      g++            \
      autoconf       \
      linux-headers  \
      make           \
      libzip-dev     \
      icu-dev        \
      postgresql-dev \
 # PHP Extensions --------------------------------- \
  && curl -sSLf \
        -o /usr/local/bin/install-php-extensions \
        https://github.com/mlocati/docker-php-extension-installer/releases/download/2.1.54/install-php-extensions \
  && chmod +x /usr/local/bin/install-php-extensions \
  && install-php-extensions gd \
  && docker-php-ext-configure pgsql -with-pgsql=/usr/local/pgsql \
  && docker-php-ext-install -j$(nproc) \
      intl        \
      opcache     \
      zip         \
      pdo_pgsql   \
 # Pecl Extensions --------------------------------- \
  && pecl install apcu && docker-php-ext-enable apcu \
 # --------------------------------------------------\
 # Install Xdebug at this step to make editing dev image cache-friendly, we delete xdebug from production image later \
  && pecl install xdebug-${XDEBUG_VERSION} \
 # Cleanup ----------------------------------------------------------------------------------------------------------- \
  && rm -r /tmp/pear; \
 # - Detect Runtime Dependencies of the installed extensions. \
 # - src: https://github.com/docker-library/wordpress/blob/master/latest/php8.2/fpm-alpine/Dockerfile \
    out="$(php -r 'exit(0);')"; \
    	[ -z "$out" ]; err="$(php -r 'exit(0);' 3>&1 1>&2 2>&3)"; \
    	[ -z "$err" ]; extDir="$(php -r 'echo ini_get("extension_dir");')"; \
    	[ -d "$extDir" ]; \
    	runDeps="$( \
    		scanelf --needed --nobanner --format '%n#p' --recursive "$extDir" \
    			| tr ',' '\n' | sort -u \
    			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    	)"; \
        # Save Runtime Deps in a virtual deps
    	apk add --no-cache --no-network --virtual .php-extensions-rundeps $runDeps; \
        # Uninstall Everything we Installed (minus the runtime Deps)
    	apk del --no-network .build-deps; \
    	\
    	! { ldd "$extDir"/*.so | grep 'not found'; }; \
        # check for output like "PHP Warning:  PHP Startup: Unable to load dynamic library 'foo' (tried: ...)
    	err="$(php --version 3>&1 1>&2 2>&3)"; [ -z "$err" ]
# -------------------------------------------------------------------------------------------------------------------- \

# ------------------------------------------------- Permissions --------------------------------------------------------

# - Clean bundled config/users & recreate them with UID 1000 for docker compatability in dev container.
# - Create composer directories (since we run as non-root later)
# - Add Default Config
RUN deluser --remove-home www-data && adduser -u1000 -D www-data && rm -rf /var/www /usr/local/etc/php-fpm.d/* && \
    mkdir -p /var/www/.composer /app && chown -R www-data:www-data /app /var/www/.composer; \
# ------------------------------------------------ PHP Configuration ---------------------------------------------------
    mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

# Add in Base PHP Config
COPY docker/app/php/base-*   $PHP_INI_DIR/conf.d

# ---------------------------------------------- PHP FPM Configuration -------------------------------------------------

# PHP-FPM config
COPY docker/app/fpm/*.conf  /usr/local/etc/php-fpm.d/

# --------------------------------------------------- Scripts ----------------------------------------------------------

COPY docker/app/entrypoint/*-base docker/app/post-build/*-base docker/app/pre-run/*-base \
     docker/app/fpm/healthcheck-fpm		\
     docker/app/scripts/command-loop*	\
     # to
     /usr/local/bin/

RUN  chmod +x /usr/local/bin/*-base /usr/local/bin/healthcheck-fpm /usr/local/bin/command-loop*

# ---------------------------------------------------- Composer --------------------------------------------------------

COPY --from=composer /usr/bin/composer /usr/bin/composer

# ----------------------------------------------------- MISC -----------------------------------------------------------

WORKDIR /app
USER www-data

# Common PHP Frameworks Env Variables
ENV APP_ENV prod
ENV APP_DEBUG 0
ENV TZ=Europe/Moscow

# Validate FPM config (must use the non-root user)
RUN php-fpm -t

# ---------------------------------------------------- HEALTH ----------------------------------------------------------

HEALTHCHECK CMD ["healthcheck-fpm"]

# -------------------------------------------------- ENTRYPOINT --------------------------------------------------------

ENTRYPOINT ["entrypoint-base"]
CMD ["php-fpm"]

# ======================================================================================================================
#                                                  --- Vendor ---
# ---------------  This stage will install composer runtime dependinces and install app dependinces.  ------------------
# ======================================================================================================================

FROM composer as vendor

ARG PHP_VERSION
ARG COMPOSER_AUTH
ARG APP_BASE_DIR

# A Json Object with remote repository token to clone private Repos with composer
# Reference: https://getcomposer.org/doc/03-cli.md#composer-auth
ENV COMPOSER_AUTH $COMPOSER_AUTH

WORKDIR /app

# Copy Dependencies files
COPY $APP_BASE_DIR/composer.json composer.json
COPY $APP_BASE_DIR/composer.lock composer.lock

# Set PHP Version of the Image
RUN composer config platform.php ${PHP_VERSION}; \
    # Install Dependencies
    composer install -n --no-progress --ignore-platform-reqs --no-dev --prefer-dist --no-scripts --no-autoloader

# ======================================================================================================================
# ==============================================  PRODUCTION IMAGE  ====================================================
#                                                   --- PROD ---
# ======================================================================================================================

FROM base AS app

ARG APP_BASE_DIR
USER root

# Copy PHP Production Configuration
COPY docker/app/php/prod-*   $PHP_INI_DIR/conf.d/

# Copy Prod Scripts && delete xdebug
COPY docker/app/entrypoint/*-prod docker/app/post-build/*-prod docker/app/pre-run/*-prod \
     # to
     /usr/local/bin/

RUN  chmod +x /usr/local/bin/*-prod && pecl uninstall xdebug

USER www-data

# ----------------------------------------------- Production Config -----------------------------------------------------

# Copy Vendor
COPY --chown=www-data:www-data --from=vendor /app/vendor /app/vendor

# Copy App Code
COPY --chown=www-data:www-data $APP_BASE_DIR/ .

## Run Composer Install again
## ( this time to run post-install scripts, autoloader, and post-autoload scripts using one command )
RUN composer install --optimize-autoloader --apcu-autoloader --no-dev -n --no-progress && \
    composer check-platform-reqs && \
    post-build-base && post-build-prod

ENTRYPOINT ["entrypoint-prod"]
CMD ["php-fpm"]

# ======================================================================================================================
# ==============================================  DEVELOPMENT IMAGE  ===================================================
#                                                   --- DEV ---
# ======================================================================================================================

FROM base as app-dev

ENV APP_ENV dev
ENV APP_DEBUG 1

# Switch root to install stuff
USER root

# For Composer Installs
RUN apk --no-cache add git openssh bash; \
 # Enable Xdebug
 docker-php-ext-enable xdebug

ARG XDEBUG_IDE_KEY

ENV XDEBUG_CLIENT_HOST="host.docker.internal"
ENV XDEBUG_IDE_KEY=$XDEBUG_IDE_KEY
# ---------------------------------------------------- Scripts ---------------------------------------------------------

# Copy Dev Scripts
COPY docker/app/php/dev-*   $PHP_INI_DIR/conf.d/
COPY docker/app/entrypoint/*-dev  docker/app/post-build/*-dev docker/app/pre-run/*-dev \
     # to
     /usr/local/bin/

RUN chmod +x /usr/local/bin/*-dev; \
    mv "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini"

USER www-data
# ------------------------------------------------- Entry Point --------------------------------------------------------

# Entrypoints
ENTRYPOINT ["entrypoint-dev"]
CMD ["php-fpm"]
