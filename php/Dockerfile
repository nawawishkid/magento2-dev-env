FROM magento/magento-cloud-docker-php:7.4-fpm-1.3.0

RUN curl -O https://download.libsodium.org/libsodium/releases/libsodium-1.0.18.tar.gz \
        && tar xfvz libsodium-1.0.18.tar.gz \
        && cd libsodium-1.0.18 \
        && ./configure \
        && make && make install \
        && pecl install -f libsodium

RUN mv /usr/local/etc/php/php.ini-production /usr/local/etc/php/php.ini \
	&& sed -i 's/;extension=sodium/extension=sodium/g' /usr/local/etc/php/php.ini

RUN apt-get update -y \
	&& apt-get install -y cron \
	&& service cron start

COPY --from=composer:1 /usr/bin/composer /usr/bin/composer

RUN composer config -g repo.packagist composer https://packagist.org \
    && composer global require hirak/prestissimo -vvv

COPY ./etc/php-fpm.conf /usr/local/etc/php-fpm.conf
COPY ./etc/php-fpm.ini /usr/local/etc/php/conf.d/zz-magento.ini
COPY ./docker-entrypoint.sh /docker-entrypoint.sh
RUN ["chmod", "+x", "/docker-entrypoint.sh"]
ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ["php-fpm", "-R"]
