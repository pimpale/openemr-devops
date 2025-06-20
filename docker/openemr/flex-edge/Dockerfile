FROM alpine:edge AS base

#Install dependencies and fix issue in apache
RUN apk --no-cache upgrade
RUN apk add --no-cache \
    apache2 apache2-ssl apache2-utils apache2-proxy git php84 php84-tokenizer php84-ctype php84-session php84-apache2 \
    php84-json php84-pdo php84-pdo_mysql php84-curl php84-ldap php84-openssl php84-iconv \
    php84-xml php84-xsl php84-gd php84-zip php84-soap php84-mbstring php84-zlib \
    php84-mysqli php84-sockets php84-xmlreader php84-redis perl php84-simplexml php84-xmlwriter php84-phar php84-fileinfo \
    php84-sodium php84-calendar php84-intl php84-opcache php84-pecl-apcu php84-fpm \
    mariadb-client mariadb-connector-c tar curl imagemagick nodejs npm \
    certbot openssl git openssl-dev dcron \
    rsync shadow jq ncurses \
    && sed -i 's/^Listen 80$/Listen 0.0.0.0:80/' /etc/apache2/httpd.conf
# Needed to ensure permissions work across shared volumes with openemr, nginx, and php-fpm dockers
    RUN usermod -u 1000 apache
#Stuff for developers since this predominantly a developer/tester docker
RUN apk add --no-cache \
    unzip vim nano bash bash-doc bash-completion tree

#BELOW LINE NEEDED TO SUPPORT PHP8 ON ALPINE 3.13+; SHOULD BE ABLE TO REMOVE THIS IN FUTURE ALPINE VERSIONS
RUN cp /usr/bin/php84 /usr/bin/php
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer

# TODO: Note that flex series 3.14+ needs to keep build-base package in (ie. not apk del it after done) for now
#  since npm package libxmljs needs to be built during openemr build; this is part of the ccda npm build and
#  can place build-base in below apk del line when this issue is fixed)
#  btw, when this is fixed and we apk del build-base (actually, will not even need to install it),
#     this will decrease size of docker by 190MB :)
RUN apk add --no-cache build-base \
    && mkdir -p /var/www/localhost/htdocs/openemr/sites \
    && chown -R apache /var/www/localhost/htdocs/openemr \
    && mkdir -p /etc/ssl/certs /etc/ssl/private \
    # && apk del --no-cache build-base \
    && sed -i 's/^ *CustomLog/#CustomLog/' /etc/apache2/httpd.conf \
    && sed -i 's/^ *ErrorLog/#ErrorLog/' /etc/apache2/httpd.conf \
    && sed -i 's/^ *CustomLog/#CustomLog/' /etc/apache2/conf.d/ssl.conf \
    && sed -i 's/^ *TransferLog/#TransferLog/' /etc/apache2/conf.d/ssl.conf
WORKDIR /var/www/localhost/htdocs
VOLUME [ "/etc/letsencrypt/", "/etc/ssl" ]
#configure apache & php properly
ENV APACHE_LOG_DIR=/var/log/apache2
COPY php.ini /etc/php84/php.ini
COPY openemr.conf /etc/apache2/conf.d/
#add runner and auto_configure and prevent auto_configure from being run w/o being enabled
COPY openemr.sh ssl.sh xdebug.sh auto_configure.php /var/www/localhost/htdocs/
COPY utilities/unlock_admin.php utilities/unlock_admin.sh /root/
RUN chmod 500 openemr.sh ssl.sh xdebug.sh /root/unlock_admin.sh \
    && chmod 000 auto_configure.php /root/unlock_admin.php
#fix issue with apache2 dying prematurely
RUN mkdir -p /run/apache2
#Copy dev tools alias to root and create snapshots and certs dir
COPY utilities/devtools /root/
COPY utilities/devtoolsLibrary.source /root/
RUN mkdir /snapshots
RUN mkdir /certs
RUN mkdir -p /couchdb/original
#Copy demo data to root
COPY utilities/demo_5_0_0_5.sql /root/
RUN chmod 500 /root/devtools
#Ensure swarm/orchestration pieces are available if needed
RUN mkdir /swarm-pieces \
    && rsync --owner --group --perms --delete --recursive --links /etc/ssl /swarm-pieces/
#go
CMD [ "./openemr.sh" ]

EXPOSE 80 443


# kcov coverage build target
FROM base AS kcov

# Install kcov dependencies
RUN apk add --no-cache bash \
                       cmake \
                       binutils-dev \
                       curl-dev \
                       elfutils \
                       elfutils-dev \
                       g++ \
                       libcurl \
                       libdwarf-dev \
                       libelf-static \
                       pkgconfig \
                       python3

# Install kcov from source
RUN cd /tmp && \
    git clone https://github.com/SimonKagstrom/kcov && \
    cd kcov && \
    mkdir build && \
    cd build && \
    cmake .. && \
    make && \
    make install

# Create kcov wrapper script
COPY kcov-wrapper.sh /var/www/localhost/htdocs/
RUN chmod 500 /var/www/localhost/htdocs/kcov-wrapper.sh

# Create directory for coverage reports
RUN mkdir -p /var/www/localhost/htdocs/coverage

# Use kcov wrapper as entrypoint
CMD [ "./kcov-wrapper.sh" ]


# Put this last because we don't want kcov in the default.
FROM base AS final
