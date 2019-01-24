FROM debian:jessie

LABEL maintainer="deniska.info@gmail.com"

ENV PHP_INI_DIR /etc/php5/apache2
ENV GPG_KEYS 0B96609E270F565C13292B24C13C70B87267B52D 0A95E9A026542D53835E3F3A7DEC4E69FC9C83D7 0E604491
ENV PHP_VERSION 5.3.29

## Install persistent / runtime deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    mc \
    mc-data \
    htop \
    ca-certificates \
    sudo \
    ssmtp \
    zip \
## phpize deps
    autoconf \
    file \
    g++ \
    gcc \
    make \
    pkg-config \
    re2c \
    autoconf2.13 \
    curl \
    libcurl4-openssl-dev \
    libc-dev \
    libc-client-dev \
    libc-client2007e \
    libc-client2007e-dev \
    libbz2-dev \
    libfreetype6 \
    libfreetype6-dev \
    libjpeg-dev \
    libkrb5-dev \
    libmagickwand-dev \
    libmagickwand-6.q16-dev \
    libmysqlclient-dev \
    libreadline6-dev \
    librecode0 \
    librecode-dev \
    libsqlite3-0 \
    libsqlite3-dev \
    libssl-dev \
    libxml2 \
    libxml2-dev \
    libpq-dev \
    libpng-dev \
    libzip-dev \
    libmcrypt-dev \
    xz-utils \
    w3m \
    zlib1g-dev \
## apache2
    apache2-bin \
    apache2-dev \
    apache2.2-common && \
    rm -rf /var/www/html && \
    mv /etc/apache2/apache2.conf /etc/apache2/apache2.conf.dist && \
    mkdir -p /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www/docroot && \
    mkdir -p $PHP_INI_DIR/conf.d && \
    mkdir /usr/include/freetype2/freetype && \
    ln -s /usr/include/freetype2/freetype.h /usr/include/freetype2/freetype/freetype.h && \
    chown -R www-data:www-data /var/lock/apache2 /var/run/apache2 /var/log/apache2 /var/www && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -r /var/lib/apt/lists/* && \
## Apache + PHP requires preforking Apache for best results
    a2dismod mpm_event && \
    a2enmod mpm_prefork && \
    a2enmod rewrite && \
    a2enmod env && \
    a2enmod headers && \
    a2enmod include && \
    a2enmod ssl

## Check pgp keys
RUN set -xe && \
    for key in $GPG_KEYS; do \
        gpg --keyserver ha.pool.sks-keyservers.net --recv-keys 0x"$key"; \
    done
## Checking keys may cause error - gpg: no valid OpenPGP data found
## In this case we can check several keyservers:
#    for key in $GPG_KEYS; do \
#        ( gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$key" \
#          || gpg --keyserver keyserver.pgp.com --recv-keys "$key" \
#          || gpg --keyserver keyserver.ubuntu.com --recv-keys "$key" \
#          || gpg --keyserver pgp.mit.edu --recv-keys "$key" ); \
#    done && \

## Compile openssl, otherwise --with-openssl won't work
## get the openssl version number here:
## https://www.openssl.org/news/openssl-1.0.2-notes.html
RUN CFLAGS="-fPIC" && \
    OPENSSL_VERSION="1.0.2q" && \
    cd /tmp && \
    mkdir openssl && \
    curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz" -o openssl.tar.gz && \
    curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz.asc" -o openssl.tar.gz.asc && \
    ls -la && \
    gpg --verify -vvvv openssl.tar.gz.asc && \
    tar -xzf openssl.tar.gz -C openssl --strip-components=1 && \
    cd /tmp/openssl && \
    ./config -fPIC && make && make install && make clean && \
    rm -rf /tmp/*

## PHP 5.3 build from source
RUN set -x && \
    curl -SL "https://php.net/get/php-$PHP_VERSION.tar.xz/from/this/mirror" -o php.tar.xz && \
    curl -SL "https://php.net/get/php-$PHP_VERSION.tar.xz.asc/from/this/mirror" -o php.tar.xz.asc && \
    gpg --verify php.tar.xz.asc && \
    mkdir -p /usr/src/php && \
    tar -xof php.tar.xz -C /usr/src/php --strip-components=1 && \
    rm php.tar.xz* && \
    cd /usr/src/php && \
    ./configure --disable-cgi \
        $(command -v apxs2 > /dev/null 2>&1 && echo '--with-apxs2=/usr/bin/apxs2' || true) \
        --with-config-file-path="$PHP_INI_DIR" \
        --with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
        --enable-bcmath \
        --enable-calendar \
        --enable-exif \
        --enable-ftp \
        --enable-mbstring \
        --enable-mysqlnd \
        --enable-soap \
        --enable-sockets \
        --enable-zip \
        --with-bz2 \
        --with-curl \
        --with-freetype-dir=/usr/include \
        --with-gd \
        --with-imap \
        --with-imap-ssl \
        --with-jpeg-dir=/usr/lib/x86_64-linux-gnu \
        --with-kerberos \
        --with-mysql \
        --with-mysqli \
        --with-pdo-mysql \
        --with-mhash \
        --with-openssl=/usr/local/ssl \
        --with-pgsql=/usr/local/pgsql \
        --with-pdo-pgsql \
        --with-png \
        --with-readline \
        --with-xmlrpc \
        --with-zlib && \
    make -j"$(nproc)" && \
    make install && \
    make clean

COPY apache/apache2.conf /etc/apache2/apache2.conf
COPY apache/docker-php-* /usr/local/bin/
COPY php/00-php.ini $PHP_INI_DIR/conf.d/

## Install/configure PHP extensions
RUN { find /usr/local/bin /usr/local/sbin -type f -executable -exec strip --strip-all '{}' + || true; } && \
    export PHP_EXT_DIR=$(php -r 'echo ini_get("extension_dir");') && \
    ln -s /usr/lib/x86_64-linux-gnu/ImageMagick-6.8.9/bin-Q16/MagickWand-config /usr/bin && \
    pecl install uploadprogress && \
    pecl install imagick-3.1.2 && \
    docker-php-ext-configure mcrypt && \
    docker-php-ext-install mcrypt && \
    curl -o /tmp/pdflib-php.tar.gz -SL "http://www.pdflib.com/binaries/PDFlib/804/PDFlib-8.0.4-Linux-x86_64-php.tar.gz" && \
    curl -o /tmp/tet-php.tar.gz -SL "http://www.pdflib.com/binaries/TET/430/TET-4.3p1-Linux-x86_64.tar.gz" && \
# List files from tar archive: tar -tvf PDFlib-8.0.4-Linux-x86_64-php.tar.gz |grep libpdf_php.so
# Extract specific file from tar archive, exclude directory structure (--strip-components argument)
    tar -xvf /tmp/pdflib-php.tar.gz -C $PHP_EXT_DIR PDFlib-8.0.4-Linux-x86_64-php/bind/php/php-530/libpdf_php.so --strip-components 4 && \
    tar -xvf /tmp/tet-php.tar.gz -C $PHP_EXT_DIR TET-4.3p1-Linux-x86_64/bind/php/php-530/php_tet.so --strip-components 4 && \
    echo 'extension=libpdf_php.so' >> $PHP_INI_DIR/conf.d/20-pdflib.ini && \
    echo 'extension=php_tet.so' >> $PHP_INI_DIR/conf.d/25-tet.ini && \
    echo 'extension=uploadprogress.so' >> $PHP_INI_DIR/conf.d/30-uploadprogress.ini && \
    echo 'extension=imagick.so' >> $PHP_INI_DIR/conf.d/40-imagick.ini && \
    echo 'extension=mcrypt.so' >> $PHP_INI_DIR/conf.d/50-mcrypt.ini && \
## TODO: add magickwand
#    echo 'extension="/usr/local/lib/php/extensions/magickwand.so"' >> $PHP_INI_DIR/conf.d/40-magickwand.ini
# Apache gets grumpy about PID files pre-existing
    rm -f /var/run/apache2/apache2.pid && \
    rm -rf /tmp/* && \
    rm -rf /usr/src/php

EXPOSE 80 443

WORKDIR /var/www

CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]