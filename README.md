# drupal8
Stack LEMP for Drupal 8
## Docker Configuration
Please note that this configuration is intended for local development only.  It has not been secured or optimized for production use.

1. Start with a fresh composer install.  We're going to update the basic composer.json so do the very minimal setup step.  In a terminal window in your site parent directory (say ~/sites or ~/gitrepo) run this command.

```
 composer create-project drupal/recommended-project my_site
```

2. We're going to work with a number of files in the my_site directory created by this command so ```cd my_site```

3. Create this docker-compose.yml file using your favorite editor:

```
version: '3'
services:
  db:
    image: mariadb:10.3
    environment:
      MYSQL_DATABASE: drupal
      MYSQL_ROOT_PASSWORD: MyGreatPassword
    volumes:
      - db_data:/var/lib/mysql
    restart: always
  drupal:
    depends_on:
      - db
    build: .
    ports:
      - "2345:80"
      - "2443:443"
    volumes:
      - ./:/var/www/html
    restart: always
  solr:
    image: solr:8
    ports:
      - "8983:8983"
    volumes:
      - ./mycores/collection1:/mycores/collection1
    entrypoint:
      - docker-entrypoint.sh
      - solr-precreate
      - collection1
      - /mycores/collection1
volumes:
  db_data:
```

4. Create the Dockerfile for the drupal container, which should be in the site root directory

```
FROM drupal:8-apache

RUN apt-get update && apt-get install -y --no-install-recommends \
  ssl-cert \
	curl \
	git \
	mariadb-client \
	vim \
	zip \
	unzip \
	wget

RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \
	php composer-setup.php && \
	mv composer.phar /usr/local/bin/composer && \
	php -r "unlink('composer-setup.php');" && \
	wget -O drush.phar https://github.com/drush-ops/drush-launcher/releases/download/0.4.2/drush.phar && \
	chmod +x drush.phar && \
	mv drush.phar /usr/local/bin/drush && \
	rm -rf /var/www/html/* && \
  rm -r /var/lib/apt/lists/* && \
  a2enmod ssl && \
  a2ensite default-ssl && \
  sed -i "s/^[ \t]*DocumentRoot \/var\/www\/html$/DocumentRoot \/var\/www\/html\/web/" /etc/apache2/sites-enabled/default-ssl.conf


COPY apache-drupal.conf /etc/apache2/sites-enabled/000-default.conf

WORKDIR /var/www/html/web
```

If you're paying close attention you'll notice we're using the '8-drupal' container instead of the '9-drupal' container as some of the packages we're using for this tutorial are not yet compatible with php 8.x which is used by the 9-drupal image.

7. Add the apache-drupal.conf file with the default VirtualHost

```
  ServerName drupal.localdomain
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html/web

<VirtualHost *:80>
  ServerName drupal.localdomain
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html/web

	<Directory /var/www/html/web>
		AllowOverride All
		Require all granted
	</Directory>

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

6. Start containers with

```
docker-compose up -d
```
