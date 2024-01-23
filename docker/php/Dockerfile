FROM php:8.3-fpm

RUN apt-get update && apt-get install -y libzip-dev libicu-dev

RUN docker-php-ext-install zip intl opcache mysqli

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin/ --filename=composer

WORKDIR /application

COPY . /application

RUN composer update

RUN php bin/console d:m:m -n

EXPOSE 9000
