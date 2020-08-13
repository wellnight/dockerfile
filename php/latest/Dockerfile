FROM wellnight/php:fpm7.4.9-alpine3.11

# 安装composer
RUN curl -sS https://getcomposer.org/installer | php \
    && mv composer.phar /usr/local/bin/composer \
    && composer config -g repo.packagist composer https://packagist.phpcomposer.com \
    && composer self-update --clean-backups \
    # 依赖项
    && apk add --update --no-cache --virtual .build-deps autoconf build-base \
    # 安装扩展
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install opcache \
    # pecl安装扩展
    && pecl install xdebug \
    # 配置
    && docker-php-ext-enable xdebug \
    # 清空缓存
    && apk del .build-deps \
    && rm -rf /usr/share/man /tmp/* /var/cache/apk/*

CMD ["php-fpm"]
