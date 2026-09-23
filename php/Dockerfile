FROM php:5.6-apache

# Debian stretch is EOL — repoint apt at the archive mirror so packages resolve
RUN sed -i \
        -e 's|deb.debian.org/debian|archive.debian.org/debian|g' \
        -e 's|security.debian.org/debian-security|archive.debian.org/debian-security|g' \
        -e '/stretch-updates/d' \
        /etc/apt/sources.list

# Build deps for gd (png/jpeg) and zip
RUN apt-get -o Acquire::Check-Valid-Until=false update \
    && apt-get install -y --no-install-recommends --allow-unauthenticated \
        libpng-dev \
        libjpeg-dev \
        libzip-dev \
        zlib1g-dev \
    && docker-php-ext-configure gd --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install mysqli mysql gd mbstring zip \
    && rm -rf /var/lib/apt/lists/*

# phpBB needs URL rewriting and .htaccess overrides
RUN a2enmod rewrite \
    && sed -ri 's/AllowOverride None/AllowOverride All/g' /etc/apache2/apache2.conf

EXPOSE 80
