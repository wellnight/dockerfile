FROM alpine:3.11

# dependencies required for running "phpize"
# these get automatically installed and removed by "docker-php-ext-*" (unless they're already installed)
ENV PHPIZE_DEPS \
	autoconf \
	dpkg-dev dpkg \
	file \
	g++ \
	gcc \
	libc-dev \
	make \
	pkgconf \
	re2c

ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64" 

ENV PHP_INI_DIR=/usr/local/etc/php \
	PHP_CPPFLAGS="$PHP_CFLAGS" \
	PHP_LDFLAGS="-Wl,-O1 -pie" \
	PHP_VERSION=7.4.9

ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=www-data --with-fpm-group=www-data --disable-cgi

# persistent / runtime deps
RUN set -eux; \
	sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories; \
	apk update; \
	apk add --no-cache \
	ca-certificates \
	curl \
	tar \
	xz \
	# https://github.com/docker-library/php/issues/494
	autoconf \
	openssl \
	tzdata; \
	# time
	cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
	echo "Asia/Shanghai" > /etc/timezone; \
	apk del tzdata; \
	# ensure www-data user exists
	# 82 is the standard uid/gid for "www-data" in Alpine
	# https://git.alpinelinux.org/aports/tree/main/apache2/apache2.pre-install?h=3.9-stable
	# https://git.alpinelinux.org/aports/tree/main/lighttpd/lighttpd.pre-install?h=3.9-stable
	# https://git.alpinelinux.org/aports/tree/main/nginx/nginx.pre-install?h=3.9-stable
    addgroup -g 82 -S www-data; \
    adduser -u 82 -D -S -G www-data www-data; \
	\
	mkdir -p "$PHP_INI_DIR/conf.d"; \
	# allow running as an arbitrary user (https://github.com/docker-library/php/issues/743)
	[ ! -d /var/www/html ]; \
	mkdir -p /var/www/html; \
	chown www-data:www-data /var/www/html; \
	chmod 777 /var/www/html; \
	\
	mkdir -p /usr/src; \
	cd /usr/src 

COPY php-7.4.9.tar.xz /usr/src/php.tar.xz

COPY docker-php-* /usr/local/bin/

RUN set -eux; \
	apk add --no-cache --virtual .build-deps \
	$PHPIZE_DEPS \
	argon2-dev \
	coreutils \
	curl-dev \
	libedit-dev \
	libsodium-dev \
	libxml2-dev \
	linux-headers \
	oniguruma-dev \
	openssl-dev \
	rabbitmq-c-dev \
	sqlite-dev; \
	\
	export CFLAGS="$PHP_CFLAGS" \
	CPPFLAGS="$PHP_CPPFLAGS" \
	LDFLAGS="$PHP_LDFLAGS" \
	; \
	docker-php-source extract; \
	cd /usr/src/php; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	./configure \
	--build="$gnuArch" \
	--with-config-file-path="$PHP_INI_DIR" \
	--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
	\
	# make sure invalid --configure-flags are fatal errors instead of just warnings
	--enable-option-checking=fatal \
	\
	# https://github.com/docker-library/php/issues/439
	--with-mhash \
	--enable-sockets \
    --enable-bcmath \
    --without-sqlite3 \
	\
	# --enable-ftp is included here because ftp_ssl_connect() needs ftp to be compiled statically (see https://github.com/docker-library/php/issues/236)
	--enable-ftp \
	# --enable-mbstring is included here because otherwise there's no way to get pecl to use it properly (see https://github.com/docker-library/php/issues/195)
	--enable-mbstring \
	# --enable-mysqlnd is included here because it's harder to compile after the fact than extensions are (since it's a plugin for several extensions, not an extension in itself)
	--enable-mysqlnd \
    --with-pdo-mysql=mysqlnd \
	# https://wiki.php.net/rfc/argon2_password_hash (7.2+)
	--with-password-argon2 \
	# https://wiki.php.net/rfc/libsodium
	--with-sodium=shared \
	--with-curl \
	--with-libedit \
	--with-openssl \
	--with-zlib \
	\
	# in PHP 7.4+, the pecl/pear installers are officially deprecated (requiring an explicit "--with-pear") and will be removed in PHP 8+; see also https://github.com/docker-library/php/issues/846#issuecomment-505638494
	--with-pear \
	\
	# bundled pcre does not support JIT on s390x
	# https://manpages.debian.org/stretch/libpcre3-dev/pcrejit.3.en.html#AVAILABILITY_OF_JIT_SUPPORT
	$(test "$gnuArch" = 's390x-linux-musl' && echo '--without-pcre-jit') \
	\
	${PHP_EXTRA_CONFIGURE_ARGS:-} \
	; \
	make -j "$(nproc)"; \
	find -type f -name '*.a' -delete; \
	make install; \
	find /usr/local/bin /usr/local/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; \
	make clean; \
	\
	# https://github.com/docker-library/php/issues/692 (copy default example "php.ini" files somewhere easily discoverable)
	cp -v php.ini-* "$PHP_INI_DIR/"; \
	\
	cd /; \
	docker-php-source delete; \
	\
	runDeps="$( \
	scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
	| tr ',' '\n' \
	| sort -u \
	| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --no-cache $runDeps; \
	\
	# update pecl channel definitions https://github.com/docker-library/php/issues/443
	pecl update-channels; \
	rm -rf /tmp/pear ~/.pearrc; \
	# smoke test
	php --version; \
    pecl install redis amqp; \
	apk del --no-network .build-deps; \
	# sodium was built as a shared module (so that it can be replaced later if so desired), so let's enable it too (https://github.com/docker-library/php/issues/598)
	docker-php-ext-enable sodium redis amqp

ENTRYPOINT ["docker-php-entrypoint"]
##<autogenerated>##
WORKDIR /var/www/html

RUN set -eux; \
	cd /usr/local/etc; \
	if [ -d php-fpm.d ]; then \
	# for some reason, upstream's php-fpm.conf.default has "include=NONE/etc/php-fpm.d/*.conf"
	sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
	cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
	else \
	# PHP 5.x doesn't use "include=" by default, so we'll create our own simple config that mimics PHP 7+ for consistency
	mkdir php-fpm.d; \
	cp php-fpm.conf.default php-fpm.d/www.conf; \
	{ \
	echo '[global]'; \
	echo 'include=etc/php-fpm.d/*.conf'; \
	} | tee php-fpm.conf; \
	fi; \
	{ \
	echo '[global]'; \
	echo 'error_log = /proc/self/fd/2'; \
	echo; echo '; https://github.com/docker-library/php/pull/725#issuecomment-443540114'; echo 'log_limit = 8192'; \
	echo; \
	echo '[www]'; \
	echo '; if we send this to /proc/self/fd/1, it never appears'; \
	echo 'access.log = /proc/self/fd/2'; \
	echo; \
	echo 'clear_env = no'; \
	echo; \
	echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
	echo 'catch_workers_output = yes'; \
	echo 'decorate_workers_output = no'; \
	} | tee php-fpm.d/docker.conf; \
	{ \
	echo '[global]'; \
	echo 'daemonize = no'; \
	echo; \
	echo '[www]'; \
	echo 'listen = 9000'; \
	} | tee php-fpm.d/zz-docker.conf

# Override stop signal to stop process gracefully
# https://github.com/php/php-src/blob/17baa87faddc2550def3ae7314236826bc1b1398/sapi/fpm/php-fpm.8.in#L163
STOPSIGNAL SIGQUIT

EXPOSE 9000
CMD ["php-fpm"]
##</autogenerated>##
