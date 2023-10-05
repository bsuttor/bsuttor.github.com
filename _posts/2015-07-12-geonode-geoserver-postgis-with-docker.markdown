---
layout: post
title: "Geonode, Geoserver, Postgis with Docker"
date: 2015-07-12 21:37:27 +0200
tags: Python Buildout Django Geonode Postgis Postgres Geoserver DevOps Docker
---

*We have some clients which needs a framework for maps creation. We took a look on market of open source solutions for this kind of feature and we became fan of geonode project. We begin to be familiar with Docker so we decided to create Docker images for Geonode. We would like to separate geoserver and geonode. The goal is to be able to move geoserver or geonode on distinct server if the load increase. So we create different images for Geonode, Geoserver. We also use Nginx image for creation of link between Geonode and Geoserver (and a Postgis image for development).*

## Intro
We have some clients which needs a framework for maps creation. We took a look on market of open source solutions for this kind of feature and we became fan of geonode project.

We begin to be familiar with Docker so we decided to create Docker images for Geonode. We would like to separate geoserver and geonode. The goal is to be able to move geoserver or geonode on distinct server if the load increase. So we create different images for Geonode, Geoserver. We also use Nginx image for creation of link between Geonode and Geoserver (and a Postgis image for development).

![Geonode](img/geonode.png)

For this project, we customise geonode. We use django template for project creation as explain on documentation ( http://docs.geonode.org/en/latest/tutorials/devel/projects/setup.html).

```bash
$ django-admin startproject imio_geonode --template=https://github.com/GeoNode/geonode-project/archive/master.zip -epy,rst
```

## docker-compose
https://docs.docker.com/compose

We use 2 differents docker-compose.yml files, one for production and one for development.

Differences are :

- entrypoint and command options are define in Dockerfile for prod and in docker-compose.yml for dev (we overide options on dev).
- image vs build : build is used in dev. For production, we build images and use image option.
- postgis : An image of postgis is used in dev and no image on prod, we use a specific database cluster.

This is development docker-compose.yml
```
postgis:
 build: Dockerfiles/postgis/
 hostname: postgis
 volumes:
 - ./postgres_data:/var/lib/postgresql

geoserver:
 build: Dockerfiles/geoserver/
 hostname: geoserver
 links:
 - postgis
 ports:
 - 8080:8080
 volumes:
 - ./geoserver_data:/opt/geoserver/data_dir

geonode:
 build: .
 hostname: geonode
 links:
 - postgis
 ports:
 - 8000:8000
 volumes:
 - .:/opt/geonode/
 entrypoint:
 - /usr/bin/python
 command: manage.py runserver 0.0.0.0:8000

nginx:
 image: nginx:latest
 ports:
 - 80:80
 links:
 - geonode
 - geoserver
 - postgis
 volumes:
 - nginx-default.conf:/etc/nginx/conf.d/default.conf
```

## Nginx
We need and nginx images to make the link between geoserver and geonode. With Docker >= 1.8 and docker-compose >= 1.4, a new 'network' option arrived and seems to depreced this nginx utility.

Nginx default image (https://registry.hub.docker.com/_/nginx/) is used with this config:
```
upstream geonode {
    server geonode:8000;
}
upstream geoserver {
    server geoserver:8080;
}

server {
        listen   80;
        client_max_body_size 128m;

        location / {
            proxy_pass         http://geonode;
            proxy_set_header   Host $http_host;
            proxy_set_header   X-Real-IP       $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /geoserver {
            proxy_pass         http://geoserver;
            proxy_set_header   Host $http_host;
            proxy_set_header   X-Real-IP       $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        error_log /var/log/nginx/error.log warn;
        access_log /var/log/nginx/access.log combined;
}
```
Nginx is useful for connection login from geoserver to geonode and for upload layers, ... from geonode to geoserver.

For production, we add static and upload folder rules :

```
location /static {
 alias /opt/geonode/static;
 }
location /uploaded {
 alias /opt/geonode/uploaded;
 }
```
## Postgis
We use postgis docker image for development. Indead, we have a dedicated server for our databases. For dev, we use this Dockerfile :

```
FROM postgres:9.4

RUN apt-get update && apt-get install -y postgresql-9.4-postgis-2.1

RUN mkdir /docker-entrypoint-initdb.d
COPY ./initdb-postgis.sh /docker-entrypoint-initdb.d/initdb-postgis.sh
```

From postgres images, all sh files in docker-entrypoint-initdb.d folder are run during postgres initialisation. And db init script for geonode looks like :
```
#!/bin/sh
POSTGRES="gosu postgres"

$POSTGRES postgres --single -E <<EOSQL
CREATE ROLE geonode ENCRYPTED PASSWORD 'geonode' LOGIN;
EOSQL
$POSTGRES postgres --single -E <<EOSQL
CREATE DATABASE geonode OWNER geonode ;
CREATE DATABASE "geonode-imports" OWNER geonode ;
EOSQL
$POSTGRES pg_ctl -w start
$POSTGRES psql -d geonode-imports -c 'CREATE EXTENSION postgis;'
$POSTGRES psql -d geonode-imports -c 'GRANT ALL ON geometry_columns TO PUBLIC;'
$POSTGRES psql -d geonode-imports -c 'GRANT ALL ON spatial_ref_sys TO PUBLIC;'
```

## Geoserver
I create an Docker image from 'tomcat:8-jre7' image, and install geoserver from http://build.geonode.org/geoserver/latest/geoserver.war.

Dockerfile looks like :
```
FROM tomcat:8-jre7

RUN apt-get update && apt-get install wget
RUN wget -O /usr/local/tomcat/webapps/geoserver.war http://build.geonode.org/geoserver/latest/geoserver.war
RUN apt-get remove -y wget

ENV GEOSERVER_DATA_DIR /opt/geoserver/data_dir
```

## Geonode
### Production
We use gunicorn for production (https://pypi.python.org/pypi/gunicorn/)

### Development
We use 'python manage.py runserver' for development. As you see in docker-compose.yml file, source code is added into a docker image with a volume, thus when you change code on your local computer, it's directly update on docker image.

Dockerfile:
```
FROM ubuntu:14.04

RUN \
 apt-get update && \
 apt-get install -y build-essential && \
 apt-get install -y libxml2-dev libxslt1-dev libjpeg-dev gettext git python-dev python-pip libgdal1-dev && \
 apt-get install -y python-pillow python-lxml python-psycopg2 python-django python-bs4 python-multipartposthandler transifex-client python-paver python-nose python-django-nose python-gdal python-django-pagination python-django-jsonfield python-django-extensions python-django-taggit python-httplib2

RUN mkdir -p /opt/geonode

WORKDIR /opt/geonode
ADD requirements.txt /opt/geonode/
RUN pip install -r requirements.txt
ADD . /opt/geonode


COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
```

## Settings and local_settings
You have to update your local_settings. Settings have to match with settings into Dockerfiles and docker-compose.yml. Here is an exemple of local_settings.py for OGC_SERVER and DATABASES : 

```
SITENAME = 'GeoNode'
SITEURL = 'http://localhost'

GEOSERVER_URL = SITEURL + '/geoserver/'
GEOSERVER_BASE_URL = GEOSERVER_URL
# OGC (WMS/WFS/WCS) Server Settings
OGC_SERVER = {
 'default': {
 'BACKEND': 'geonode.geoserver',
 'LOCATION': 'http://172.17.42.1:80/geoserver/', # Docker IP
 'PUBLIC_LOCATION': GEOSERVER_URL,
 'USER': 'admin',
 'PASSWORD': 'admin',
 'MAPFISH_PRINT_ENABLED': True,
 'PRINT_NG_ENABLED': True,
 'GEONODE_SECURITY_ENABLED': True,
 'GEOGIT_ENABLED': False,
 'WMST_ENABLED': False,
 'BACKEND_WRITE_ENABLED': True,
 'WPS_ENABLED': True,
 # Set to name of database in DATABASES dictionary to enable
 'DATASTORE': 'datastore',
 }
}

DATABASES = {
 'default': {
 'ENGINE': 'django.db.backends.postgresql_psycopg2',
 'NAME': 'geonode',
 'USER': 'geonode',
 'PASSWORD': 'geonode',
 'HOST': 'postgis',
 'PORT': 5432,
 },
 'datastore': {
 'ENGINE': 'django.contrib.gis.db.backends.postgis',
 'NAME': 'geonode-imports',
 'USER': 'geonode',
 'PASSWORD': 'geonode',
 'HOST': 'postgis',
 'PORT': 5432,
 }
}
```

You also have to set link between Geoserver and Geonode login. Use Docker IP for settings that (geoserver_data/security/auth/geonodeAuthProvider/config.xml) :
```
<org.geonode.security.GeoNodeAuthProviderConfig>
 <id>-0000000000000</id>
 <name>geonodeAuthProvider</name>
 <className>org.geonode.security.GeoNodeAuthenticationProvider</className>
 <baseUrl>http://172.17.42.1:80/</baseUrl>
</org.geonode.security.GeoNodeAuthProviderConfig>
```

## Conclusion
You can now start you geonode project with a simple :
```
$ docker-compose up
```
And make Django syncdb of your database :
```
$ docker-compose run --rm --entrypoint='/usr/bin/python' geonode manage.py syncdb
```