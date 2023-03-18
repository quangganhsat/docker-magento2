# Docker Magento2: Varnish 7 + PHP 8.1 + Redis 6.2 + Elasticsearch 7.17 + SSL cluster ready docker-compose infrastructure

## Infrastructure overview
* Container 1: Mysql 8.0
* Container 2: Redis 7.0 (volatile, for Magento's cache)
* Container 3: Redis 7.0 (for Magento's sessions)
* Container 4: Apache 2.4 + PHP 8.2 (modphp)
* Container 5: Cron
* Container 6: Varnish 7.1
* Container 7: Redis (volatile, cluster nodes autodiscovery)
* Container 8: Nginx SSL terminator
* Container 9: Elasticsearch 7.17

### Why a separate cron container?
First of all containers should be (as far as possible) single process, but the most important thing is that (if someday we'll be able to deploy this infrastructure in production) we may need a cluster of apache+php containers but a single cron container running.

Plus, with this separation, in the context of a docker swarm, you may be able in the future to separare resources allocated to the cron container from the rest of the infrastructure.

## Setup Magento 2

Download Magento 2 in any way you want (zip/tgz from website, composer, etc) and extract in the "magento2" subdirectory of this project.

If you want to change the default "magento2" directory simply change its name in the "docker-compose.yml" (there are 2 references, under the "cron" section and under the "apache" section).

## Starting all docker containers
```
docker-compose up -d
```
The fist time you run this command it's gonna take some time to download all the required images from docker hub.

## Install Magento2

### Method 1: CLI
```
docker exec -it docker-magento2-apache-1 bash
php bin/magento setup:install \
  --db-host docker-magento2-db-1 --db-name magento2 --db-user magento2 --db-password magento2  --admin-user admin --timezone 'Europe/Rome' --currency EUR --use-rewrites 1 --cleanup-database \
  --backend-frontname admin --admin-firstname AdminFirstName --admin-lastname AdminLastName --admin-email 'admin@email.com' --admin-password 'ChangeThisPassword1' --base-url 'https://magento2.docker/' --language en_US \
  --session-save=redis --session-save-redis-host=sessions --session-save-redis-port=6379 --session-save-redis-db=0 --session-save-redis-password='' \
  --cache-backend=redis --cache-backend-redis-server=cache --cache-backend-redis-port=6379 --cache-backend-redis-db=0 \
  --page-cache=redis --page-cache-redis-server=cache --page-cache-redis-port=6379 --page-cache-redis-db=1 \
  --search-engine=elasticsearch7 --elasticsearch-host=elasticsearch
```

## Deploy static files
```
docker exec -it docker-magento2-apache-1 bash
php bin/magento dev:source-theme:deploy
php bin/magento setup:static-content:deploy
```

## Enable Varnish
Varnish Full Page Cache should already be enabled out of the box (we startup Varnish with the default VCL file generated by Magento2) but you could anyway go to "stores -> configuration -> advanced -> system -> full page cache" and:
* select Varnish in the "caching application" combobox
* type "apache" in both "access list" and "backend host" fields
* type 80 in the "backend port" field
* save

Configure Magento to purge Varnish:

```
docker exec -it docker-magento2_apache_1 bash
php bin/magento setup:config:set --http-cache-hosts=varnish
```

https://devdocs.magento.com/guides/v2.3/config-guide/varnish/use-varnish-cache.html

## Enable SSL Support
Add this line to magento2/.htaccess
```
SetEnvIf X-Forwarded-Proto https HTTPS=on
```
Then you can configure Magento as you wish to support secure urls.

If you need to generate new self signed certificates use this command
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```
then you can mount them into the nginx-ssl container using the "volumes" instruction in the docker-compose.xml file. Same thing goes if you need to use custom nginx configurations (you can mount them into /etc/nginx/conf.d). Check the source code of https://github.com/fballiano/docker-nginx-ssl-for-magento2 to better understand where are the configuration stored inside the image/container.

## Scaling apache containers
If you need more horsepower you can
```
docker-compose scale apache=X
```
where X is the number of apache containers you want.

The cron container will check how many apache containers we have (broadcast/discovery service is stored on the redis_clusterdata container) and will update Varnish's VCL.

You can start your system with just one apache container, then scale it afterward, autodiscovery will reconfigure the load balancing on the fly.

Also, the cron container (which updates Varnish's VCL) sets a "probe" to "/fb_host_probe.txt" every 5 seconds, if 1 fails (container has been shut down) the container is considered sick.

## Custom php.ini
We already have a personalized php.ini inside this project: https://github.com/fballiano/docker-magento2-apache-php/blob/master/php.ini but if you want to further customize your settings:
- edit the php.ini file in the root directoy of this project
- edit the "docker-compose.xml" file, look for the 2 commented lines (under the "cron" section and under the "apache" section) referencing the php.ini
- start/restart the docker stack

Please note that your php.ini will be the last parsed thus you can ovverride any setting.

## Tested on:
* Docker for Mac 19

## TODO
* DB clustering?
* RabbitMQ?
let me know what features would you like to see implemented.

## Changelog:
* 2020-09-19:
  * Magento 2.4 branch added
  * Elasticsearch container added for Magento 2.4
  * Upaded all dependencies
* 2020-03-18:
  * added "sockets" PHP extension to docker-apache-php image
  * fixed some typos/mistakes in the README
  * added CLI install to the README
  * refactored some parts of the documentation to better use magento's CLI
* 2019-08-09:
  * small bugfix in varnishadm.sh and varnishncsa.sh scripts
* 2019-08-06:
  * new redis sessions container was added
* 2019-08-05:
  * migrated to docker-compose syntax 3.7
  * implemented "delegated" consistency for some of volumes for a better performance
  * varnish.vcl was regenerated for Varnish 5 (which was already used since some months)
